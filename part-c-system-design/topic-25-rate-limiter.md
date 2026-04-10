# Topic 25: Design a Rate Limiter (Distributed)

## Concept
A system that restricts client requests to a maximum rate (e.g., 100 requests/minute per user). Prevents abuse, protects backend, ensures fair resource sharing. Challenges: distributed deployment (multiple servers), accuracy at scale, minimal latency overhead.

## Simple Explanation
Rate limiter is like a restaurant host:
- **Reservation limit:** Only seat 10 parties/hour (rate limit)
- **Queue management:** First come, first served (FIFO)
- **Fairness:** One party can't book all tables (per-user limit, not global)
- **Burst handling:** A party of 10 counts as 1 but blocks smaller tables (token bucket vs fixed window)
- **Distributed problem:** Multiple hosts taking reservations → need shared ledger (Redis) to coordinate

---

## System Design: Step-by-Step

### STEP 1: Clarify Requirements

**Functional:**
- Limit user to N requests per M seconds
- Return 429 Too Many Requests when exceeded
- Support multiple limit levels (global, per-user, per-IP, per-endpoint)

**Non-Functional:**
- **Accuracy:** Very accurate for 1-2% of requests (tail behavior), approximate for high volume OK
- **Latency:** Add < 1ms overhead per request
- **Availability:** Rate limiter failure should NOT block traffic (fail-open)
- **Distributed:** Work across multiple servers/regions

**Examples:**
- GitHub API: 60 requests/hour (authenticated), 10 requests/hour (anonymous)
- Stripe: 100 requests/second per account
- Twitter: 450 requests/15-minute window per app

### STEP 2: Estimate Scale

```
Users: 100M
API servers: 100 (for load distribution)
Rps per server: 10,000 (10K requests/sec)
Rate limit checks: 100M users × average 10 req/day = 1.2M rate limit checks/sec

Example scenario:
- Endpoint: POST /api/post/create
- Limit: 10 posts per 60 seconds per user
- Peak: 10,000 users creating posts concurrently
  Rate limit checks: 10,000 * 60 sec window = 600K state updates/sec
```

### STEP 3: High-Level Architecture

```
                          ┌─ Redis Cluster
                          │ (tracks request counts)
                          │ Key: "rate:user:123"
                          │ Value: count
                          │
API Request ──→ LB ──→ API Server ──→ Rate Limiter ──→ If OK: Continue processing
(with user ID)             |              (< 1ms)         If blocked: Return 429
                          │                              │
                          └─ Fallback: in-memory         └─ Analytics queue
                             (if Redis unavailable)         (track abuse patterns)

Rate Limiter Libraries:
- Token Bucket
- Sliding Window
- Leaky Bucket
```

### STEP 4: Deep-Dive: Algorithms

#### Algorithm 1: Fixed Window (Simple but Flawed)

```java
// Problem: Burst attacks at window boundary
// Time: 00:00 ─── 01:00 ─── 02:00 (each window = 1 minute)
// Limit: 10 requests per minute

// Request timeline at 00:59:
@Transactional
public boolean isAllowed(String userId, int limit) {
    String key = "rate:" + userId;
    long count = redis.get(key);  // Get current window count
    
    if (count >= limit) {
        return false;  // Reject: already hit limit
    }
    
    redis.increment(key);  // Increment counter
    redis.expire(key, 60);  // Reset TTL to 60 seconds
    return true;
}

❌ PROBLEM: Boundary attack
Time: 00:59:50 - User makes 10 requests (hits limit at 00:59:59)
Time: 01:00:00 - Window resets, user immediately makes 10 more requests
Result: 20 requests in 10 seconds (should be 10/min!)

// User makes requests at:
// 00:59:50, 00:59:51, ..., 00:59:59 (10 requests)
// 01:00:00, 01:00:01, ..., 01:00:09 (10 requests)
// Total: 20 requests in 20 seconds (not 10/min!)
```

#### Algorithm 2: Sliding Window (Accurate but Expensive)

```java
// ✅ CORRECT: Accurate rate limiting (no boundary attacks)
// Cost: Higher computational overhead

public boolean isAllowed(String userId, int limit, long windowSeconds) {
    String key = "rate:sliding:" + userId;
    long now = System.currentTimeMillis();
    
    // 1. Remove old requests outside the window
    redis.zremrangebyscore(key, 0, now - windowSeconds * 1000);
    
    // 2. Count requests in current window
    long count = redis.zcard(key);  // Count elements in sorted set
    
    if (count >= limit) {
        return false;  // Reject
    }
    
    // 3. Add current request with timestamp
    redis.zadd(key, now, UUID.randomUUID().toString());
    
    // 4. Set TTL (after window, no cleanup needed)
    redis.expire(key, windowSeconds + 1);
    
    return true;
}

// Data structure in Redis (sorted set):
// key: "rate:sliding:user:123"
// members: [
//   (score=1707000000000, member="uuid-1"),
//   (score=1707000500000, member="uuid-2"),
//   ...
// ]
// Score = timestamp, members = request IDs (for uniqueness)

Example timeline (10 requests/60 seconds, now = 1707000900000):
Request at 00:59:50 (score=1707000800000)
Request at 00:59:55 (score=1707000830000)
Request at 01:00:00 (score=1707000900000)  ← Now
...

When new request arrives:
1. Remove all requests with score < (1707000900000 - 60000) = 1707000840000
2. Count remaining requests
3. If < 10, add new request
Result: Exact sliding window, no boundary attacks!

❌ PROBLEM: O(n) computation per request (expensive for large windows)
```

#### Algorithm 3: Token Bucket (Best for Latency)

```java
// ✅ RECOMMENDED: Token bucket combines simplicity and accuracy
// Allows burst, prevents sustained overload, O(1) operation

public class TokenBucket {
    private final int capacity;        // Max tokens
    private final int refillRate;      // Tokens per second
    private double tokens;
    private long lastRefillTime;
    
    public TokenBucket(int capacity, int refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = capacity;
        this.lastRefillTime = System.currentTimeMillis();
    }
    
    public synchronized boolean isAllowed(int tokensRequired) {
        // 1. Refill tokens based on time passed
        long now = System.currentTimeMillis();
        long elapsedMs = now - lastRefillTime;
        
        double refilled = (elapsedMs / 1000.0) * refillRate;
        tokens = Math.min(capacity, tokens + refilled);
        lastRefillTime = now;
        
        // 2. Check if enough tokens available
        if (tokens >= tokensRequired) {
            tokens -= tokensRequired;  // Consume tokens
            return true;
        }
        
        return false;
    }
}

// Example: 10 req/min = 1 req/6 seconds
// Capacity = 10, RefillRate = 10/60 = 0.167 tokens/sec

Timeline:
T=00:00 - tokens=10, request consumes 1, tokens=9
T=00:30 - 30 sec passed, 30 * 0.167 = 5 new tokens → tokens = min(10, 9+5) = 10 (refilled!)
Result: Smooth, fair rate limiting + allows short bursts

In Redis (distributed):
```java
public boolean isAllowedDistributed(String userId, int capacity, int refillRate) {
    String key = "bucket:" + userId;
    
    // Redis Lua script (atomic operation)
    String script = """
        local capacity = tonumber(ARGV[1])
        local refillRate = tonumber(ARGV[2])
        local tokensRequired = tonumber(ARGV[3])
        local now = tonumber(ARGV[4])
        
        local data = redis.call('HGETALL', KEYS[1])
        local tokens = tonumber(data[2]) or capacity
        local lastRefill = tonumber(data[4]) or now
        
        local elapsed = (now - lastRefill) / 1000.0
        local refilled = elapsed * refillRate
        tokens = math.min(capacity, tokens + refilled)
        
        if tokens >= tokensRequired then
            tokens = tokens - tokensRequired
            redis.call('HSET', KEYS[1], 'tokens', tokens, 'lastRefill', now)
            redis.call('EXPIRE', KEYS[1], 3600)
            return 1
        end
        
        return 0
        """;
    
    Object result = redis.eval(script, Collections.singletonList(key),
        capacity, refillRate, 1, System.currentTimeMillis());
    
    return (Long) result == 1;
}
```

### STEP 5: Implementation Patterns

#### Pattern 1: Per-User Rate Limiting (Most Common)

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) throws Exception {
        String userId = extractUserId(request);  // From JWT or session
        String endpoint = request.getRequestURI();
        
        // Different limits per endpoint
        int limit = getLimit(endpoint);  // e.g., 10 posts/min, 100 reads/min
        
        String key = "rate:" + userId + ":" + endpoint;
        
        // Token bucket check
        if (!isAllowed(key, limit)) {
            response.setStatus(429);  // Too Many Requests
            response.getWriter().write("""
                {
                  "error": "Rate limit exceeded",
                  "retry_after": 60
                }
                """);
            return false;  // Block request
        }
        
        return true;  // Allow request
    }
    
    private boolean isAllowed(String key, int limit) {
        try {
            // Use Lua script for atomicity
            return executeTokenBucket(key, limit);
        } catch (Exception e) {
            log.error("Rate limiter failure", e);
            return true;  // Fail-open: allow request if rate limiter is down
        }
    }
}
```

#### Pattern 2: Global Rate Limiting (API Gateway Level)

```java
@Configuration
public class RateLimiterConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("api_post", r -> r
                .path("/api/posts/**")
                .filters(f -> f
                    .requestRateLimiter(c -> c
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver())
                    )
                )
                .uri("http://postservice"))
            .build();
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(
            10,      // replenish rate: 10 tokens/second
            20       // capacity: max 20 tokens (burst)
        );
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest()
                .getHeaders()
                .getFirst("X-User-Id")
        );
    }
}
```

#### Pattern 3: Resilience4j Rate Limiter (Fallback)

```java
@Service
public class PaymentService {
    
    @RateLimiter(name = "payment", fallbackMethod = "fallback")
    public PaymentResult processPayment(Payment payment) {
        // Attempt payment
        return externalPaymentGateway.charge(payment);
    }
    
    // Fallback: reject gracefully if rate limit hit
    public PaymentResult fallback(Payment payment, Exception e) {
        log.warn("Rate limit exceeded, queuing payment for retry", e);
        
        // Queue for later processing instead of failing
        paymentQueue.add(payment);
        
        return PaymentResult.builder()
            .status(QUEUED)
            .message("Payment queued, will be processed shortly")
            .build();
    }
}
```

### STEP 6: Bottlenecks & Solutions

#### Bottleneck 1: Redis Hotspot

```
Problem: All rate limiter calls go to same Redis shard
→ Redis bottleneck (single connection pool, network saturation)

Scenario:
- 100K users × 10 requests per minute = 16.6K rate limit checks/sec
- All users stored in same Redis instance
- Result: Redis CPU maxes out, API servers timeout on rate limiter check

Solution: Redis Cluster with consistent hashing

// Sharding: Different users → different Redis nodes
key = "rate:" + userId + ":" + endpoint
shard = hash(key) % 10  // Distributes across 10 Redis nodes

User 1 → Redis Node 1
User 2 → Redis Node 2
...
User 100K → Redis Node 7

Result: Load distributed 10x, each node handles ~1.6K checks/sec (manageable)
```

#### Bottleneck 2: Network Latency to Redis

```
Problem: Every request requires Redis round-trip
→ 1-5ms per request (significant for < 100ms SLA)

Scenario:
- API latency target: 50ms
- Redis latency: 5ms (network + server)
- Rate limiter + response serialization: 40ms
- Request processing: 20ms
- Total: 5 + 40 + 20 = 65ms (exceeds SLA!)

Solution: Local cache with eventual consistency

@Autowired
private LoadingCache<String, TokenBucket> localCache;

public boolean isAllowed(String key) {
    // Check local cache first (< 1ms)
    TokenBucket bucket = localCache.getUnchecked(key);
    
    if (bucket.isAllowed(1)) {
        return true;  // Allow immediately
    }
    
    // Only go to Redis if local bucket rejected
    // Chance: 1-2% (when user is close to limit)
    return redis.isAllowed(key);  // Double-check with authoritative source
}

Result:
- 98% of requests serviced in < 1ms (local cache)
- 2% pay 5ms penalty (Redis check)
- Average: ~0.1ms overhead
- Eventual consistency: Local counter might be 1-2% off, acceptable
```

#### Bottleneck 3: Distributed Coordination

```
Problem: Multiple API servers have different local state
→ User makes 5 concurrent requests across 5 different servers
→ Each server checks local cache, all pass limit
→ Effective limit becomes 5x larger than intended

Scenario:
User limit: 10 requests/minute
Request 1 hits Server A: local cache says allowed
Request 2 hits Server B: local cache says allowed (different server!)
Request 3 hits Server C: local cache says allowed
...
Result: User makes 50 requests in parallel (5x limit overshot!)

Solution: Combined local + Redis validation

```java
public boolean isAllowed(String userId, int limit) {
    // 1. Local check (optimistic, low latency)
    int localCount = localCounter.get(userId, 0);
    if (localCount >= limit) {
        // Might be stale, verify with Redis
        int redisCount = redis.get(userId);
        if (redisCount >= limit) {
            return false;  // Confirmed: hit limit
        }
        // Redis says OK, update local (eventual consistency)
        localCounter.set(userId, redisCount);
        localCount = redisCount;
    }
    
    // 2. Increment both local and Redis
    localCounter.increment(userId);
    redis.increment(userId);  // Async/non-blocking
    
    return true;
}

Result:
- Local cache gives < 1ms latency
- Redis provides authoritative count (eventual consistency)
- Short-term overages (1-2 requests) acceptable
- Long-term accuracy guaranteed
```

---

## Real Project Usage

**Project:** Fintech payment platform protecting against brute-force attacks

**Attacks Observed:**
1. Credential stuffing: 1000 login attempts/second from single IP
2. Replay attacks: Attacker replaying captured tokens
3. Distributed attacks: Bot farm from 1000 IPs, each staying under per-IP limit

**Solution Implemented:**

1. **Per-IP Rate Limit:** 10 failed logins/minute (blocks credential stuffing)
   ```java
   String ipKey = "rate:login:failed:" + clientIp;
   if (!isAllowed(ipKey, 10)) {
       // Return 429, block IP for 1 minute
       blockIp(clientIp, 60_000);
   }
   ```

2. **Per-User Rate Limit:** 5 failed logins/minute (blocks user enumeration)
   ```java
   String userKey = "rate:login:failed:" + username;
   if (!isAllowed(userKey, 5)) {
       // Lock account temporarily
       lockAccount(username, 300_000);  // 5 minutes
   }
   ```

3. **Global Rate Limit:** 1K login attempts/sec (circuit breaker)
   ```java
   String globalKey = "rate:login:global";
   if (!isAllowed(globalKey, 1000)) {
       // Under DDoS, enable CAPTCHA
       enableCaptcha = true;
   }
   ```

4. **Behavioral Deviation Detection:** Use Redis to track unusual patterns
   ```
   User normally logs in 2 times/day
   Suddenly 100 attempts in 1 hour → trigger MFA challenge
   ```

**Results:**
- Reduced successful credential stuffing by 99.8%
- DDoS impact mitigated (IP-level blocking)
- False positive rate: 0.01% (legitimate users rarely hit limits)

---

## Interview Answer (2–3 Minutes)

> "For a distributed rate limiter, I'd use a token bucket algorithm with Redis backend.
>
> **Why token bucket?** Because it's simple (O(1) operation), allows controlled bursts (important for user experience), and scales well.
>
> **Design:** Each user has a bucket with N tokens. Every second, we add R tokens (up to capacity). Each request consumes 1 token. If no tokens left, return 429.
>
> **Distributed:** Use Redis to store authoritative bucket state (tokens, last refill time). Lua script for atomicity — single Redis call retrieves state, refills, checks availability, and updates — no race conditions.
>
> **Performance optimization:** Local cache with eventual consistency. 98% of requests check local memory (< 1ms). Only on cache misses or when user is close to limit, we hit Redis. Acceptable overages: 1-2 requests. Eventual consistency ensures accuracy over time.
>
> **Scaling:** Redis cluster with consistent hashing. Each user hashes to a shard, distributing load across 10+ nodes instead of one.
>
> **Resilience:** If Redis goes down, fall back to local-only rate limiting with degraded accuracy. Better to allow some requests than block all traffic.
>
> **Different limits per endpoint:** POST /tweet = 10/min, GET /timeline = 100/min. Configuration-driven, no code changes needed.
>
> The key trade-off: **Accuracy vs latency.** Perfect accuracy would require synchronous Redis call per request (5ms latency). I sacrifice 1-2% accuracy for < 1ms latency using local cache."

**Follow-up likely:** "What if an attacker tries to bypass rate limiting via distributed requests from 1000 IPs?" → Implement global rate limit (per endpoint), behavioral analysis (abnormal patterns), or require CAPTCHA on failure.

---

## Quick Revision Card — Rate Limiter

| Component | Choice | Trade-off |
|---|---|---|
| **Algorithm** | Token Bucket | Simple, burst-friendly, O(1) |
| **Storage** | Redis + local cache | Accuracy vs latency |
| **Scope** | Per-user + per-IP + global | Layered defense |
| **Consistency** | Eventual (eventual accuracy) | Lower latency, temporary overages OK |
| **Failure Mode** | Fail-open (allow if down) | Better UX than blocking everything |
| **Concurrency** | Lua script (atomic) | Race condition prevention |
| **Scalability** | Redis Cluster + sharding | Distribute load across nodes |
| **Monitoring** | Track rate limit rejections | Detect attacks, DDoS patterns |

---

**End of Topic 25**
