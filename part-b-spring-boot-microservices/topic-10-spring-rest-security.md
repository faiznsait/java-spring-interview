# Topic 10: Spring REST, Security, and API Design

## Q51. GET vs POST vs PUT — REST Method Differences and GET Limitations

### Concept
HTTP methods define the **intent** of a request. REST APIs use them to signal whether you're reading, creating, or replacing a resource.

### Comparison Table

| | `GET` | `POST` | `PUT` |
|---|---|---|---|
| **Purpose** | Read / retrieve | Create new resource | Replace/update existing resource |
| **Body** | ❌ No body (params in URL) | ✅ Body | ✅ Body |
| **Idempotent** | ✅ (same request = same result) | ❌ | ✅ |
| **Safe** | ✅ (no side effects) | ❌ | ❌ |
| **Cacheable** | ✅ | ❌ | ❌ |
| **Bookmarkable** | ✅ | ❌ | ❌ |

### GET Limitations
```java
// 1. URL length limit (~2000 chars for older browsers/servers)
//    Can't send large payloads — use POST for search with complex filters

// 2. Params exposed in URL — visible in browser history, server logs, Referer header
GET /api/users?password=secret123  // NEVER do this — password in URL!

// 3. No request body — do not rely on bodies with GET; many servers/proxies reject/ignore them
//    Use query params for retrieval filters, or POST when body semantics are needed

// 4. Not suitable for state changes — GET must be idempotent and have no side effects

// ─── Spring REST ──────────────────────────────────────────────────────────────
@RestController
@RequestMapping("/api/users")
public class UserController {

    // GET — read, idempotent, cacheable
    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // POST — create, NOT idempotent (calling twice creates two resources)
    @PostMapping
    public ResponseEntity<UserDTO> createUser(@Valid @RequestBody CreateUserRequest req) {
        UserDTO created = userService.create(req);
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest().path("/{id}")
            .buildAndExpand(created.getId()).toUri();
        return ResponseEntity.created(location).body(created);  // 201 Created
    }

    // PUT — replace entire resource, idempotent (calling twice = same state)
    @PutMapping("/{id}")
    public ResponseEntity<UserDTO> updateUser(@PathVariable Long id,
                                              @Valid @RequestBody UpdateUserRequest req) {
        return ResponseEntity.ok(userService.replace(id, req));
    }

    // PATCH — partial update (not mentioned in Q but worth knowing)
    @PatchMapping("/{id}")
    public ResponseEntity<UserDTO> patchUser(@PathVariable Long id,
                                             @RequestBody Map<String, Object> updates) {
        return ResponseEntity.ok(userService.patch(id, updates));
    }

    // DELETE — remove resource, idempotent
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();  // 204 No Content
    }
}
```

### HTTP Status Codes — Quick Reference
```
200 OK            — successful GET, PUT, PATCH
201 Created       — successful POST (include Location header)
204 No Content    — successful DELETE
400 Bad Request   — validation failure, malformed JSON
401 Unauthorized  — not authenticated
403 Forbidden     — authenticated but lacks permission
404 Not Found     — resource doesn't exist
409 Conflict      — duplicate create, version conflict
422 Unprocessable — semantically invalid input
500 Internal      — unhandled server error
```

### Interview Answer
> "GET retrieves without side effects — it's safe and idempotent, so browsers cache it and search engines index it. Its main limitations are URL length (around 2000 chars), visibility — params appear in browser history and server logs, so you never put credentials or sensitive data in a GET — and no body. POST creates a new resource and is not idempotent — submitting twice creates two records. PUT replaces the entire resource and is idempotent — the same PUT twice leaves the same state. For partial updates I use PATCH. I always return 201 with a Location header on POST, 204 on DELETE, and 404 when a resource isn't found."
>
> *Likely follow-up: "What is the difference between PUT and PATCH?"*

---

## Q52. Is REST Synchronous or Asynchronous? — Handling High Traffic

### Concept
REST over HTTP is **synchronous by default** — the client waits for the server response. High concurrency is handled through thread pools, async processing, and reactive paradigms.

### Default Synchronous Flow
```
Client → POST /orders → Server
Client WAITS (thread blocked) ...
Server processes → 200 OK → Client UNBLOCKS
```

### Making REST Asynchronous in Spring

#### Option 1: `@Async` + `CompletableFuture`
```java
// Controller returns CompletableFuture — Servlet container doesn't block the request thread
@RestController
public class OrderController {

    @PostMapping("/orders")
    public CompletableFuture<ResponseEntity<Order>> createOrder(@RequestBody OrderRequest req) {
        return orderService.processAsync(req)
            .thenApply(order -> ResponseEntity.status(HttpStatus.CREATED).body(order));
    }
}

@Service
public class OrderService {
    @Async("orderTaskExecutor")  // Uses dedicated thread pool, not HTTP worker pool
    public CompletableFuture<Order> processAsync(OrderRequest req) {
        Order order = processHeavyWork(req);
        return CompletableFuture.completedFuture(order);
    }
}
```

#### Option 2: Accept and Poll Pattern (true async UX)
```java
// Client gets an immediate 202 Accepted, polls for result
@PostMapping("/orders")
public ResponseEntity<Void> submitOrder(@RequestBody OrderRequest req) {
    String jobId = orderQueue.submit(req);   // Queue the work, return immediately
    URI statusUri = URI.create("/orders/jobs/" + jobId);
    return ResponseEntity.accepted()         // 202 Accepted
        .header("Location", statusUri.toString())
        .build();
}

@GetMapping("/orders/jobs/{jobId}")
public ResponseEntity<JobStatus> checkStatus(@PathVariable String jobId) {
    JobStatus status = jobTracker.get(jobId);
    if (status.isDone()) return ResponseEntity.ok(status);
    return ResponseEntity.ok(status);  // Return IN_PROGRESS with percentage
}
```

#### Option 3: Spring WebFlux (Reactive — for thousands of concurrent connections)
```java
@RestController
public class ReactiveOrderController {

    @PostMapping("/orders")
    public Mono<ResponseEntity<Order>> createOrder(@RequestBody Mono<OrderRequest> reqMono) {
        return reqMono
            .flatMap(orderService::createReactive)  // Non-blocking all the way
            .map(order -> ResponseEntity.status(201).body(order));
    }
}
// Mono<T> = 0 or 1 async result
// Flux<T> = 0 to N async stream
// Uses event loop (Netty) — one thread handles thousands of connections via non-blocking I/O
```

### Handling Thousands of Concurrent Requests
```java
// Spring MVC default: Tomcat thread pool handles requests
// Default: 200 threads (spring.embedded.tomcat.threads.max)
// Tune in application.yml:
server:
  tomcat:
    threads:
      max: 400          # Max worker threads
      min-spare: 10     # Idle threads kept alive
    accept-count: 100   # Queue size when all threads busy
    connection-timeout: 20000

// Additional strategies:
// 1. Horizontal scaling + load balancer (most impactful)
// 2. Caching — Redis for frequent reads — reduces DB load
// 3. async processing for slow operations (Kafka, @Async)
// 4. Circuit breaker (Resilience4j) — fail fast when downstream is slow
// 5. WebFlux for I/O-bound high-concurrency scenarios
```

### Interview Answer
> "REST over HTTP is synchronous by default — the client blocks waiting for a response. For high throughput I use multiple strategies: increase the Tomcat thread pool for moderate load; for long-running tasks I return 202 Accepted immediately, process via a message queue, and let the client poll a status endpoint; for extreme I/O-bound concurrency I'd consider Spring WebFlux — instead of 200 threads blocking on I/O, an event loop handles thousands of connections concurrently with a handful of threads. Horizontal scaling behind a load balancer is the most cost-effective answer — add more instances rather than tuning one server to its limit."
>
> *Likely follow-up: "What is the difference between Spring MVC and Spring WebFlux?"*

---

## Q53. Authentication vs Authorization in Spring Security

### Concept
- **Authentication:** Who are you? — verify identity (username/password, JWT token)
- **Authorization:** What can you do? — verify permissions after identity is confirmed

### Simple Explanation
Authentication is the bouncer checking your ID at the door. Authorization is the different rooms inside — your ID gets you in, but you need a VIP wristband for the VIP lounge.

### How Spring Security Validates Without Controller Code

```
Request → Filter Chain (Spring Security Filters run BEFORE your controller)
            ↓
     SecurityContextPersistenceFilter  (restore SecurityContext from session)
            ↓
     UsernamePasswordAuthenticationFilter  (for form login)
     OR
     BearerTokenAuthenticationFilter  (for JWT — our most common case)
            ↓
     AuthorizationFilter  (checks @PreAuthorize / HttpSecurity rules)
            ↓
     Your Controller (only reached if all filters pass)
```

```java
// Spring Security config (Spring Boot 3 / Spring Security 6 — no WebSecurityConfigurerAdapter)
@Configuration
@EnableMethodSecurity  // Enables @PreAuthorize, @PostAuthorize, @Secured
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthFilter) {
        this.jwtAuthFilter = jwtAuthFilter;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)          // Disable for stateless REST APIs
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))  // No HTTP session
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()             // Public endpoints
                .requestMatchers("/api/admin/**").hasRole("ADMIN")       // ADMIN only
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()  // Public read
                .anyRequest().authenticated()                            // Everything else: must login
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();  // Always hash passwords with bcrypt
    }
}
```

### How Validation Works Without Controller Code
```java
// The JWT filter runs BEFORE any controller:
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
                                    throws ServletException, IOException {

        // 1. Extract token from Authorization header
        String header = request.getHeader("Authorization");
        if (header == null || !header.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);  // No token — continue (will fail auth check later)
            return;
        }

        String token = header.substring(7);

        // 2. Validate token (signature + expiry)
        if (jwtService.isValid(token)) {
            String username = jwtService.extractUsername(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            // 3. Create Authentication object and set in SecurityContext
            UsernamePasswordAuthenticationToken auth =
                new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
            auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            SecurityContextHolder.getContext().setAuthentication(auth);
            // ↑ This is what Spring Security reads to allow the request through
        }

        filterChain.doFilter(request, response);  // Pass to next filter / controller
    }
}
```

### Interview Answer
> "Authentication confirms identity — in REST APIs usually via JWT in the Authorization header. Authorization determines what that identity is allowed to do. Spring Security handles both entirely in its filter chain — no code in the controller. The `JwtAuthenticationFilter` runs before every request: it extracts the token, validates the signature and expiry, loads the user's authorities, and sets them in the `SecurityContextHolder`. The `AuthorizationFilter` further down the chain reads those authorities and checks them against `HttpSecurity` rules or method-level `@PreAuthorize` annotations. If either check fails, Spring Security returns 401 or 403 before the request ever reaches the controller."
>
> *Likely follow-up: "What happens if SecurityContextHolder is empty when the request reaches the authorization check?"*

---

## Q54. Spring Security — High-Level Flow, Filters, and Authorization Rules

### The Security Filter Chain in Detail

```
HTTP Request
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Spring Security Filter Chain (DelegatingFilterProxy)                       │
│                                                                             │
│  1. SecurityContextPersistenceFilter  → restore context from storage       │
│  2. LogoutFilter                      → handle /logout                     │
│  3. UsernamePasswordAuthFilter        → form login (if enabled)            │
│  4. BasicAuthenticationFilter         → HTTP Basic auth (if enabled)       │
│  5. BearerTokenAuthenticationFilter  → JWT / OAuth2 bearer token  ←YOU    │
│  6. SessionManagementFilter           → session fixation, concurrency      │
│  7. ExceptionTranslationFilter        → converts AuthException to 401/403  │
│  8. AuthorizationFilter              → checks roles/permissions  ← GATE   │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    ▼ (if all filters pass)
Your Controller
```

### Complete JWT-Based Security Implementation

```java
// ─── JWT Service ──────────────────────────────────────────────────────────────
@Service
public class JwtService {
    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration:3600000}")  // 1 hour default
    private long expirationMs;

    public String generateToken(UserDetails userDetails) {
        return Jwts.builder()
            .subject(userDetails.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expirationMs))
            .claim("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList())
            .signWith(getSigningKey(), Jwts.SIG.HS256)
            .compact();
    }

    public boolean isValid(String token) {
        try {
            Jwts.parser().verifyWith(getSigningKey()).build().parseSignedClaims(token);
            return true;
        } catch (JwtException e) {
            return false;  // Expired, tampered, wrong signature → rejected
        }
    }

    public String extractUsername(String token) {
        return Jwts.parser().verifyWith(getSigningKey()).build()
            .parseSignedClaims(token).getPayload().getSubject();
    }

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }
}

// ─── UserDetailsService — load user from DB ──────────────────────────────────
@Service
public class CustomUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getEmail())
            .password(user.getPasswordHash())  // BCrypt hashed
            .roles(user.getRoles().toArray(new String[0]))
            .build();
    }
}

// ─── Authorization Rules — Multiple Approaches ───────────────────────────────
// Approach 1: URL-based (in SecurityFilterChain — coarse-grained)
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/manager/**").hasAnyRole("ADMIN", "MANAGER")
    .requestMatchers(HttpMethod.DELETE, "/api/**").hasRole("ADMIN")
    .anyRequest().authenticated()
)

// Approach 2: Method-level (fine-grained — at service layer)
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { ... }

@PreAuthorize("hasRole('MANAGER') or #userId == authentication.principal.id")
public UserDTO getUser(Long userId) { ... }  // MANAGER or the user themselves

@PostAuthorize("returnObject.ownerId == authentication.principal.id")
public Document getDocument(Long id) { ... }  // Check AFTER method runs

// Approach 3: Programmatic
@Autowired
private SecurityContextHolder securityContextHolder;

public void sensitiveAction() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    if (!auth.getAuthorities().contains(new SimpleGrantedAuthority("ROLE_ADMIN"))) {
        throw new AccessDeniedException("Admin only");
    }
}
```

### Exception Handling for Security
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        ...
        .exceptionHandling(ex -> ex
            .authenticationEntryPoint((request, response, authException) -> {
                // Called when not authenticated (no token / bad token)
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);  // 401
                response.getWriter().write("{\"error\": \"Authentication required\"}");
            })
            .accessDeniedHandler((request, response, accessDeniedException) -> {
                // Called when authenticated but lacks permission
                response.setStatus(HttpServletResponse.SC_FORBIDDEN);    // 403
                response.getWriter().write("{\"error\": \"Access denied\"}");
            })
        )
        .build();
}
```

---

## Q55. JWT Token — Components and Validation

### Concept
A JWT (JSON Web Token) is a self-contained, cryptographically signed token — the server can verify it without hitting the database.

### Simple Explanation
A JWT is like a signed cheque from a bank — anyone who trusts the bank's signature can accept the cheque without calling the bank. The signature proves the content hasn't been tampered with.

### Three Parts of a JWT

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   ← Header    (Base64URL encoded)
.
eyJzdWIiOiJhbGljZUBleGFtcGxlLmNvbSIsInJvbGVzIjpbIlVTRVIiXSwiaWF0IjoxNzA0MDY3MjAwLCJleHAiOjE3MDQwNzA4MDB9
                                         ← Payload   (Base64URL encoded)
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
                                         ← Signature (HMAC or RSA)
```

```java
// ─── HEADER (algorithm + type) ────────────────────────────────────────────────
{
    "alg": "HS256",   // HMAC-SHA256 (symmetric) or RS256 (asymmetric — preferred in prod)
    "typ": "JWT"
}

// ─── PAYLOAD (claims) ─────────────────────────────────────────────────────────
{
    // Registered claims (standard):
    "sub": "alice@example.com",    // Subject — who the token is about
    "iat": 1704067200,             // Issued at (Unix timestamp)
    "exp": 1704070800,             // Expiration (iat + 1 hour)
    "iss": "https://api.myapp.com", // Issuer

    // Custom claims:
    "roles": ["USER", "MANAGER"],
    "userId": 42,
    "tenantId": "acme-corp"
}
// ⚠ Payload is BASE64 ENCODED, NOT ENCRYPTED — never put passwords/secrets in JWT claims!

// ─── SIGNATURE ────────────────────────────────────────────────────────────────
// HMAC-SHA256:
HMACSHA256(
    base64UrlEncode(header) + "." + base64UrlEncode(payload),
    secretKey
)
// RSA (RS256): private key signs, public key validates — better for microservices
// (each service only needs the public key; only auth server needs private key)
```

### Validation Flow
```java
// When a request arrives with Authorization: Bearer <token>

// Step 1: Split on "." → get header.payload.signature
// Step 2: Recompute HMAC(header + "." + payload, secretKey)
// Step 3: Compare with signature in token — if mismatch → TAMPERED → reject 401

// Step 4: Decode payload (Base64URL) → parse claims
// Step 5: Check exp → if current time > exp → EXPIRED → reject 401
// Step 6: Check iss → must match expected issuer
// Step 7: Extract sub and roles → create Authentication in SecurityContext

public boolean isValid(String token) {
    try {
        Jwts.parser()
            .verifyWith(getSigningKey())   // Validates signature
            .build()
            .parseSignedClaims(token);    // Also checks expiry automatically
        return true;
    } catch (ExpiredJwtException e) {
        log.warn("JWT expired");
        return false;
    } catch (UnsupportedJwtException | MalformedJwtException | SignatureException e) {
        log.warn("JWT invalid: {}", e.getMessage());
        return false;
    }
}
```

### Access Token vs Refresh Token
```java
// Access Token: short-lived (15 min to 1 hour) — used for API calls
// Refresh Token: long-lived (days/weeks) — used only to get new access tokens
// Stored server-side (Redis) so they can be revoked

@PostMapping("/auth/refresh")
public ResponseEntity<TokenResponse> refresh(@RequestBody RefreshRequest req) {
    String refreshToken = req.getRefreshToken();

    // Validate refresh token against Redis (server-side store)
    if (!tokenStore.isValid(refreshToken)) {
        return ResponseEntity.status(401).body(new TokenResponse("Refresh token invalid"));
    }

    String username = jwtService.extractUsername(refreshToken);
    UserDetails user = userDetailsService.loadUserByUsername(username);
    String newAccessToken = jwtService.generateToken(user);

    return ResponseEntity.ok(new TokenResponse(newAccessToken, refreshToken));
}
```

### Interview Answer
> "A JWT has three Base64URL-encoded parts: header (algorithm type), payload (claims — subject, expiry, roles, custom data), and signature. The signature is an HMAC of the first two parts using the secret key. Validation: recompute the HMAC and compare to the signature — any tampering of header or payload changes the hash, causing rejection. Then check expiry from the `exp` claim. Critical point: the payload is just encoded, not encrypted — never put sensitive data like passwords in claims. For microservices I prefer RS256 — the auth server signs with a private key, each service validates with the public key only, so the signing secret never leaves the auth server."
>
> *Likely follow-up: "How do you invalidate a JWT before it expires?"*
> (Answer: Maintain a token blacklist in Redis keyed by JWT ID (jti claim), or use short-lived access tokens + revocable refresh tokens.)

---

## Q56. OAuth 2.0, OpenID Connect, and Spring Security

### Concept
- **OAuth 2.0:** An authorization framework — lets a user grant a third-party app limited access to their resources WITHOUT sharing credentials.
- **OpenID Connect (OIDC):** An identity layer on top of OAuth 2.0 — adds **authentication** (who you are) via an ID token.
- **OAuth 2.0 = authorization. OIDC = authentication + authorization.**

### Simple Explanation
OAuth 2.0 is like a hotel key card system. You (the user) tell the hotel (authorization server) to give the valet (3rd-party app) a key card (access token) that only opens the parking garage, not your room. The valet never sees your master key (password).

OIDC adds: the hotel also gives the valet a card (ID token) that says "this card belongs to John Smith, room 204" — so the valet knows WHO the key belongs to.

### OAuth 2.0 Roles
```
Resource Owner    = User (who owns the data)
Client            = Your application (wants access to user's data)
Authorization Server = Issues tokens (Google, GitHub, Okta, Keycloak)
Resource Server   = API that holds the protected data (your backend)
```

### Authorization Code Flow (most secure — used for web apps)
```
1. User clicks "Login with Google"
2. App redirects to Google: GET /authorize?client_id=...&scope=openid profile email&redirect_uri=...
3. User logs in at Google, consents to scope
4. Google redirects back to app: GET /callback?code=AUTH_CODE
5. App exchanges code for tokens (server-to-server, secret never exposed):
   POST /token { code, client_id, client_secret, redirect_uri }
   ← Response: { access_token, refresh_token, id_token (OIDC), expires_in }
6. App uses access_token to call Google APIs
7. App reads id_token to get user identity (name, email, sub)
```

### Spring Security OAuth2 Configuration
```java
// ─── Resource Server (protect your API with OAuth2 tokens) ──────────────────
@Configuration
@EnableMethodSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .decoder(jwtDecoder())  // Validates incoming JWTs from auth server
                )
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .build();
    }

    @Bean
    public JwtDecoder jwtDecoder() {
        // Fetch public keys from auth server's JWKS endpoint (auto-rotates — more secure)
        return NimbusJwtDecoder.withJwkSetUri("https://auth.myapp.com/.well-known/jwks.json")
            .build();
        // OR for symmetric key (simpler, same secret on all services):
        // return NimbusJwtDecoder.withSecretKey(Keys.hmacShaKeyFor(secret.getBytes())).build();
    }
}

// ─── OAuth2 Login (SSO — Login with Google/GitHub/Okta) ─────────────────────
@Configuration
public class OAuth2LoginConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService)  // Map OAuth2 user to your User entity
                )
                .successHandler(oAuth2SuccessHandler)     // Generate your own JWT after OAuth2 login
            )
            .build();
    }
}

// application.yml — register OAuth2 providers
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: read:user, user:email
        provider:
          google:
            issuer-uri: https://accounts.google.com  # Spring auto-fetches OIDC discovery doc
```

### OIDC — ID Token vs Access Token
```java
// Access Token: opaque string or JWT used to call APIs
//   → "You have permission to access /api/profile"
//   → Short-lived (15 min)

// ID Token (OIDC only): JWT containing user identity claims
//   → "The user is alice@gmail.com, sub=1234567890"
//   → Signed by authorization server
//   → Used to establish a local session — NOT for API calls

// Typical OIDC ID Token payload:
{
    "iss": "https://accounts.google.com",
    "sub": "110248495820003",          // Stable unique user ID — never changes
    "email": "alice@gmail.com",
    "email_verified": true,
    "name": "Alice Smith",
    "picture": "https://...",
    "iat": 1704067200,
    "exp": 1704070800,
    "aud": "your-google-client-id"     // Must match your app — prevents token reuse
}
```

### Custom OAuth2 User Service — Map Google User to Your DB User
```java
@Service
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest request) throws OAuth2AuthenticationException {
        OAuth2User oAuth2User = super.loadUser(request);

        String email    = oAuth2User.getAttribute("email");
        String name     = oAuth2User.getAttribute("name");
        String provider = request.getClientRegistration().getRegistrationId();  // "google"

        // Find or create local user
        User user = userRepository.findByEmail(email)
            .orElseGet(() -> {
                User newUser = new User(email, name, provider);
                return userRepository.save(newUser);
            });

        // Return enriched principal with your app's roles
        return new CustomUserPrincipal(user, oAuth2User.getAttributes());
    }
}
```

### Interview Answer
> "OAuth 2.0 is an authorization framework — it defines how a user grants limited access to their resources to a third-party app without sharing their password. The app gets an access token scoped to specific permissions. OpenID Connect adds the identity layer — on top of OAuth 2.0, it returns an ID token (a JWT) proving *who* the user is, not just what they're allowed to do.
>
> In practice: if a user clicks 'Login with Google', the Authorization Code Flow kicks in — the app redirects to Google, user consents, Google returns an auth code, the app exchanges it server-to-server for tokens, and reads the ID token to create a session. I configure Spring Security as both an OAuth2 Resource Server (protecting my APIs with JWTs from the auth server) and an OAuth2 Login client (for SSO). The key detail interviewers often probe: the access token is for API calls, the ID token is for establishing identity — never pass the ID token to an API."
>
> *Likely follow-up: "What is PKCE, and why is it needed for mobile/SPA apps?"*
> (Answer: Proof Key for Code Exchange — prevents auth code interception attacks in public clients where a client secret can't be kept. The app generates a code_verifier, sends its hash (code_challenge) in the auth request, then proves it knows the original verifier when exchanging the code for tokens.)

---

## Quick Revision Card — Section 6

| Q | Core Point |
|---|---|
| **Q51** | GET=safe+idempotent+cached, no body, URL length limit, never sensitive data in URL. POST=create, not idempotent. PUT=replace, idempotent. Return 201+Location on POST, 204 on DELETE |
| **Q52** | REST is sync by default. Scale: tune Tomcat thread pool, `@Async` + `CompletableFuture`, 202 Accept+poll pattern, WebFlux for extreme I/O concurrency, or horizontal scaling |
| **Q53** | Authentication=who you are (filter validates JWT). Authorization=what you can do (checked by AuthorizationFilter). Both happen before controller. Set `SecurityContextHolder` in filter to prove identity |
| **Q54** | Filter chain order matters. `JwtAuthenticationFilter` before `UsernamePasswordAuthFilter`. `@PreAuthorize` for method-level. `ExceptionTranslationFilter` → 401 (unauthenticated) or 403 (forbidden) |
| **Q55** | 3 parts: Header (alg) + Payload (claims, Base64 not encrypted!) + Signature. Validate: recompute HMAC, check signature + expiry. RS256 > HS256 for microservices. Revoke via refresh token blacklist in Redis |
| **Q56** | OAuth2 = authorization (access token). OIDC = authentication (ID token) on top of OAuth2. Authorization Code Flow = most secure. Resource Server validates JWTs via JWKS URI. ID token = who you are, access token = what you can do |

---

**End of Section 6**

## Q102. REST Endpoint Naming Best Practices

### Resource-Oriented (Nouns, Not Verbs)

```
✅ GOOD (Nouns — resources)
GET    /api/orders              → List all orders
GET    /api/orders/{id}         → Get specific order
POST   /api/orders              → Create new order
PUT    /api/orders/{id}         → Replace entire order
PATCH  /api/orders/{id}         → Update partial order
DELETE /api/orders/{id}         → Delete order

❌ BAD (Verbs — actions)
GET    /api/getOrders
POST   /api/createOrder
POST   /api/updateOrder
POST   /api/deleteOrder
```

### Nested Resources
```java
// Relationships
GET    /api/users/{userId}/orders           → All orders for this user
GET    /api/users/{userId}/orders/{orderId} → Specific order for this user
POST   /api/users/{userId}/orders           → Create order for this user

// Keep nesting shallow (max 2 levels)
❌ /api/users/{userId}/orders/{orderId}/items/{itemId}/reviews/{reviewId}
✅ /api/reviews/{reviewId}  // Flat, use query params for filtering
```

### Query Parameters for Filtering, Sorting, Pagination
```java
@GetMapping("/api/orders")
public Page<OrderDto> getOrders(
        @RequestParam(required = false) String status,           // filter
        @RequestParam(required = false) Long userId,             // filter
        @RequestParam(defaultValue = "createdAt") String sortBy, // sort field
        @RequestParam(defaultValue = "desc") String sortDir,     // sort direction
        @RequestParam(defaultValue = "0") int page,              // pagination
        @RequestParam(defaultValue = "20") int size) {
    
    // GET /api/orders?status=PENDING&userId=123&sortBy=createdAt&sortDir=desc&page=0&size=20
}
```

### Versioning Strategies

#### 1. URL Path Versioning (Most Common)
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 { }

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 { }

// Pros: Visible, easy to route in API Gateway, clear
// Cons: URL changes with version
```

#### 2. Header Versioning
```java
@GetMapping(value = "/api/orders", headers = "X-API-Version=1")
public List<OrderV1> getOrdersV1() { }

@GetMapping(value = "/api/orders", headers = "X-API-Version=2")
public List<OrderV2> getOrdersV2() { }

// curl -H "X-API-Version: 2" https://api.example.com/orders
// Pros: Clean URLs
// Cons: Version not visible in URL, harder to test
```

#### 3. Content Negotiation (Accept Header)
```java
@GetMapping(value = "/api/orders", produces = "application/vnd.myapp.v1+json")
public List<OrderV1> getV1() { }

@GetMapping(value = "/api/orders", produces = "application/vnd.myapp.v2+json")
public List<OrderV2> getV2() { }

// curl -H "Accept: application/vnd.myapp.v2+json" https://api.example.com/orders
// Pros: RESTful, standard HTTP
// Cons: Complex, not widely used
```

### Best Practices Summary

```java
// 1. Use plural nouns
✅ /api/users
❌ /api/user

// 2. Use hyphens for multi-word resources
✅ /api/order-items
❌ /api/orderItems  (camelCase in URL is unconventional)

// 3. Lowercase only
✅ /api/users
❌ /api/Users

// 4. Don't include file extensions
✅ /api/users
❌ /api/users.json

// 5. Sub-resources for relationships
✅ /api/users/{userId}/orders
❌ /api/userOrders?userId={userId}

// 6. Actions that don't fit CRUD → sub-resource with verb
POST /api/orders/{id}/cancel     → Cancel order
POST /api/orders/{id}/refund     → Refund order
POST /api/users/{id}/activate    → Activate user
POST /api/passwords/reset        → Reset password

// 7. Health/metrics endpoints (not resource-oriented)
GET /actuator/health
GET /actuator/metrics
```

### Real Endpoint Design
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    
    // List with filtering, sorting, pagination
    @GetMapping
    public Page<OrderDto> listOrders(
            @RequestParam(required = false) String status,
            @RequestParam(required = false) Long userId,
            Pageable pageable) { }
    
    // Get single resource
    @GetMapping("/{id}")
    public OrderDto getOrder(@PathVariable Long id) { }
    
    // Create
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderDto createOrder(@Valid @RequestBody CreateOrderRequest req) { }
    
    // Full update
    @PutMapping("/{id}")
    public OrderDto updateOrder(@PathVariable Long id, @Valid @RequestBody UpdateOrderRequest req) { }
    
    // Partial update
    @PatchMapping("/{id}")
    public OrderDto patchOrder(@PathVariable Long id, @RequestBody Map<String, Object> updates) { }
    
    // Delete
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteOrder(@PathVariable Long id) { }
    
    // Custom action
    @PostMapping("/{id}/cancel")
    public OrderDto cancelOrder(@PathVariable Long id, @RequestBody CancelReason reason) { }
}
```

### Interview Answer
> "REST endpoints should be resource-oriented with plural nouns — `/api/orders`, not `/api/getOrders`. HTTP verbs indicate action: GET to fetch, POST to create, PUT to replace, PATCH to update, DELETE to remove. Use path parameters for resource IDs, query parameters for filtering/sorting/pagination.
>
> Versioning: I prefer URL path versioning (`/api/v1/orders`) — visible, easy to route in API Gateway, simple to test. Maintain N and N-1 versions, deprecate N-1 with 6 months notice via `Sunset` header.
>
> For actions that don't fit CRUD, use sub-resources with verbs: `POST /orders/{id}/cancel`, `POST /passwords/reset`. Keep nesting shallow (max 2 levels). Use hyphens for multi-word resources, lowercase only, no file extensions."

### Senior-Level API Review Checklist

- Resource naming clarity: can a frontend dev guess the endpoint without docs?
- Status code discipline: `201` for create, `204` for successful delete/update without body, `409` for conflict, `422` for validation semantics when applicable.
- Security defaults: every mutating endpoint protected by authentication + authorization + idempotency strategy.
- Backward compatibility: additive changes preferred; removal requires deprecation window and migration notice.

### Real Project Usage

- In high-traffic systems, teams often pair endpoint naming guidelines with contract tests so accidental breaking changes are caught in CI.
- For public APIs, introducing version headers and sunset policies prevented sudden client breakages during schema evolution.

### Common Mistakes to Avoid

- Returning `200 OK` for everything, including failed validation.
- Leaking internal entity models directly as API response models.
- Mixing security logic in controllers instead of centralizing in Spring Security configuration and method-level guards.

---

## Security Architecture Deep-Dive (Interview Critical)

### Authentication vs Authorization (Practical)

- Authentication answers: who are you?
- Authorization answers: what are you allowed to do?

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    @PreAuthorize("hasAnyRole('USER','ADMIN')")
    public String getOrder(@PathVariable Long id) {
        return "order-" + id;
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(@PathVariable Long id) {
        // admin only
    }
}
```

### Security Filter Chain Flow

Request -> `SecurityFilterChain` -> authentication filter -> `SecurityContext` set -> authorization checks -> controller.

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/actuator/health").permitAll()
                .requestMatchers(HttpMethod.DELETE, "/api/orders/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())
            .build();
    }
}
```

### JWT Integration Baseline

```java
// Typical JWT filter sketch (simplified)
public class JwtAuthFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String authHeader = request.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            // validate token, extract username/roles, build Authentication
            // SecurityContextHolder.getContext().setAuthentication(authentication);
        }
        filterChain.doFilter(request, response);
    }
}
```

### Exception Handling + CORS Baseline

```java
@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public Map<String, Object> handleAccessDenied(AccessDeniedException ex) {
        return Map.of("error", "FORBIDDEN", "message", ex.getMessage());
    }
}
```

Interview tip: explain that CORS should be controlled centrally in security/web config, not ad-hoc per controller.

---

