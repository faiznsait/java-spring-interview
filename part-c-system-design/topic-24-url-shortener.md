# Topic 24: Design a URL Shortener (like bit.ly)

## Concept
A system that converts long URLs into short, shareable links. Core challenges: generating unique keys efficiently, handling high read volume (10x write), minimizing latency, and serving globally.

## Simple Explanation
Imagine a library with 1 million books:
- **Long URL:** Full book address ("Building 5, Shelf 42C, Row 8, Book 3: War and Peace")
- **Short URL:** ISBN or catalog number ("034521")
- **Mapping:** Library catalog links ISBN → full address
- **Scale problem:** Millions of books, everyone wants to find books instantly (reads >> writes)
- **Solution:** Index (Redis), organize by frequency (popular books = hot shelf), replicate catalog

---

## System Design: Step-by-Step

### STEP 1: Clarify Requirements

**Functional:**
- Create a short link from a long URL
- Redirect from short link to original URL
- Optionally: custom short links, analytics (clicks, geolocation)

**Non-Functional:**
- **Scale:** 1M write operations/day → ~12 URLs/sec write, 120+ URLs/sec read (10:1 ratio)
- **Latency:** Redirect must happen in < 100ms (human perceived instant)
- **Availability:** 99.99% uptime (payment system reliability)
- **Durability:** Never lose a mapping once created
- **Lifespan:** Links last 5–10 years

### STEP 2: Estimate Scale

```
Users: 1M daily active
Avg short links created per user: 1
Writes per day: 1M
Writes per second: 1M / 86400 ≈ 12/sec

Reads per second: 120/sec (10x rate, links shared widely)
Peak (2x average): 240 reads/sec

Storage estimate:
- Short URL mapping: 50 million links accumulated
- Per entry: 200 bytes (short key, long URL, metadata)
- Total: 50M × 200 = 10GB (fits in memory!)
- With 3-year retention: 10GB manageable for primary DB

Bandwidth:
Reads: 240 reads/sec × 200 bytes ≈ 48KB/sec
Writes: 12 writes/sec × 500 bytes ≈ 6KB/sec
Total: ~54KB/sec (tiny — any network handles this)
```

### STEP 3: High-Level Architecture

```
                                    ┌─ Redis Cache
                                    │ (all mappings, 10GB)
                                    │ Hit rate: 99%
                                    │ Misses → DB lookup
                                    │
Shorten API ──→ LB ──→ API Servers ─┤
(POST /shorten)        (stateless)   │
                                    │  ┌─ Primary DB
                                    │  │ (PostgreSQL)
Get Redirect ──→ LB ──→ API Servers ─┴─┤
(GET /:shortkey)                       │ ┌─ Read Replica
                                       │ │ (for analytics)
                                       │ │
                                       └─┴─ Read Replica
                                           (geo-distributed)

Message Queue (Kafka)
├─ Event: new short link created
├─ Consumer: generate analytics (clicks, time range)
└─ Consumer: archive old links to cold storage
```

### STEP 4: Deep-Dive into Components

#### Database Schema

```sql
-- Primary table
CREATE TABLE urls (
    id BIGINT PRIMARY KEY,           -- Auto-incrementing ID (1B max)
    short_key VARCHAR(10) UNIQUE,    -- Encoded version of ID
    long_url VARCHAR(2048),          -- Original URL
    created_by VARCHAR(255),         -- User who created
    created_at TIMESTAMP,
    expires_at TIMESTAMP,            -- TTL for link
    click_count INT DEFAULT 0,       -- Analytics
    INDEX idx_short_key (short_key)
);

-- Analytics (denormalized, async populated)
CREATE TABLE analytics (
    short_key VARCHAR(10),
    click_date DATE,
    click_count INT,
    locations JSON,                  -- Geolocation data
    devices JSON,                    -- Mobile vs desktop
    PRIMARY KEY (short_key, click_date),
    INDEX idx_short_key (short_key)
);
```

#### Key Generation Strategy

```java
// ❌ WRONG: Random UUID → conflicts possible, no ordering
String shortKey = UUID.randomUUID().substring(0, 6);  // Random collisions?

// ❌ WRONG: Sequential but predictable → security issue (guess next link)
long sequence = ++counter;           // Users can enumerate all links?

// ✅ CORRECT: Auto-increment ID + Base62 encoding (no collisions, ordered)
public class UrlShortener {
    
    // Base62 encoding: A-Z, a-z, 0-9 = 62 characters
    // 6 characters = 62^6 ≈ 56 billion combinations (plenty!)
    private static final String BASE62 = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    
    // Convert auto-increment ID to Base62 short key
    public String encodeId(long id) {
        if (id == 0) return "0";
        
        StringBuilder sb = new StringBuilder();
        while (id > 0) {
            sb.append(BASE62.charAt((int)(id % 62)));
            id /= 62;
        }
        return sb.reverse().toString();
    }
    
    // Reverse: base62 → ID
    public long decodeKey(String shortKey) {
        long id = 0;
        for (char c : shortKey.toCharArray()) {
            id = id * 62 + BASE62.indexOf(c);
        }
        return id;
    }
    
    // Test
    public static void main(String[] args) {
        UrlShortener shortener = new UrlShortener();
        long id = 12345;
        String encoded = shortener.encodeId(id);
        System.out.println("ID 12345 → " + encoded);  // Output: "3d7"
        System.out.println("Reverse: " + shortener.decodeKey(encoded));  // Output: 12345
    }
}

// Result: bit.ly/3d7 (only 3 chars for first million users!)
```

#### Write Path (Create Short Link)

```java
@PostMapping("/shorten")
public ShortUrlResponse shorten(@RequestBody ShortenRequest request) {
    // Step 1: Validate long URL
    if (!isValidUrl(request.longUrl)) {
        throw new BadRequestException("Invalid URL");
    }
    
    // Step 2: Check if already shortened (dedup)
    String existing = cache.get("url:" + request.longUrl);
    if (existing != null) {
        return new ShortUrlResponse(existing);  // Return cached result
    }
    
    // Step 3: Insert into primary DB (get auto-increment ID)
    Url urlEntity = new Url();
    urlEntity.setLongUrl(request.longUrl);
    urlEntity.setCreatedBy(getCurrentUser());
    urlEntity.setCreatedAt(Instant.now());
    Url saved = repository.save(urlEntity);  // ID auto-generated
    
    // Step 4: Encode ID to short key
    String shortKey = urlShortener.encodeId(saved.getId());
    urlEntity.setShortKey(shortKey);
    repository.saveAndFlush(urlEntity);  // Update short_key
    
    // Step 5: Cache the mapping (write-through)
    cache.set("url:" + request.longUrl, shortKey, TTL=1_YEAR);
    cache.set("rev:" + shortKey, request.longUrl, TTL=1_YEAR);
    
    // Step 6: Async event: notify analytics queue
    eventBus.publish(new UrlCreatedEvent(shortKey, request.longUrl));
    
    // Respond immediately
    return new ShortUrlResponse("bit.ly/" + shortKey);
}
```

#### Read Path (Redirect - Most Critical)

```java
@GetMapping("/{shortKey}")
public ResponseEntity<?> redirect(@PathVariable String shortKey) {
    // Performance target: < 100ms p99
    
    // Step 1: Redis cache lookup (99% hit rate)
    // If cached, serve immediately (< 1ms)
    String longUrl = cache.get("rev:" + shortKey);
    if (longUrl != null) {
        // Track click async (don't block redirect)
        executor.submit(() -> trackClick(shortKey));
        return ResponseEntity.status(HttpStatus.MOVED_PERMANENTLY)
                .location(URI.create(longUrl))
                .build();
    }
    
    // Step 2: Cache miss (1% chance) → DB lookup
    // Primary DB might be overloaded, try read replica first
    Url url = readFromReplica(shortKey);
    
    // Fallback to primary if replica is lagging
    if (url == null) {
        url = readFromPrimary(shortKey);
    }
    
    if (url == null) {
        return ResponseEntity.notFound().build();  // 404
    }
    
    // Step 3: Cache for next time
    cache.set("rev:" + shortKey, url.getLongUrl(), TTL=1_YEAR);
    
    // Step 4: Async analytics
    executor.submit(() -> trackClick(shortKey));
    
    // Redirect
    return ResponseEntity.status(HttpStatus.MOVED_PERMANENTLY)
            .location(URI.create(url.getLongUrl()))
            .build();
}

// Async click tracking (doesn't block redirect)
private void trackClick(String shortKey) {
    try {
        // Increment click counter (batch writes to reduce DB load)
        clickQueue.add(shortKey);
        
        // Every 1000 clicks, batch update database
        if (clickQueue.size() >= 1000) {
            Map<String, Integer> aggregated = 
                clickQueue.stream()
                    .collect(Collectors.groupingBy(
                        Function.identity(), 
                        Collectors.summingInt(e -> 1)
                    ));
            
            // Single batch update instead of 1000 individual updates
            repository.batchUpdateClickCounts(aggregated);
            clickQueue.clear();
        }
    } catch (Exception e) {
        log.error("Failed to track click", e);
        // Don't fail redirect if analytics fails
    }
}
```

#### Cache Strategy

```yaml
# Cache layer configuration
spring:
  redis:
    # Primary cache: holds all URL mappings
    # 10GB dataset fits entirely in memory
    cluster:
      nodes: [redis-node-1:6379, redis-node-2:6379, redis-node-3:6379]
    cache:
      type: redis
      
    # Cache settings
    ttl: 31536000  # 1 year (don't expire, but data has infinite lifespan)
    max-entries: 50_000_000  # 50M URLs
    
    # Memory policy: evict LRU (least recently used) if memory full
    # LRU better than FIFO because hot links stay cached
    maxmemory-policy: allkeys-lru
```

#### Sharding Strategy (When Single DB is Overloaded)

```
Single PostgreSQL handles 12 writes/sec + 120 reads/sec easily
But with geographical distribution (multiple datacenters), we shard for locality

Sharding by geographic region (not by user):

┌─────────────────────┐
│ North America       │
│ PostgreSQL (East)   │
│ Redis (East)        │
└─────────────────────┘

┌─────────────────────┐
│ Europe              │
│ PostgreSQL (EU)     │
│ Redis (EU)          │
└─────────────────────┘

┌─────────────────────┐
│ Asia Pacific        │
│ PostgreSQL (AP)     │
│ Redis (AP)          │
└─────────────────────┘

Router logic:
```java
String getDatacenter(String longUrl) {
    if (longUrl.contains("amazon.com")) return "US";
    if (longUrl.contains("bbc.co.uk")) return "EU";
    if (longUrl.contains("baidu.com")) return "ASIA";
    return "US";  // default
}
```

Within each region, use consistent hashing if needed (rare for this scale).

### STEP 5: Bottlenecks & Solutions

#### Bottleneck 1: Viral Link

```
Problem: Drake posts a link on Twitter
→ 1M clicks in 1 second (instead of normal 120 clicks/sec)
→ Redis cache handles it (all in memory, 100K reads/sec per node)
→ But click tracking queue overflows

Solution:
- Add more Redis replicas (read-only) in each region (transparent to API)
- Use probabilistic click tracking: track 1% of clicks exactly, extrapolate
  ```java
  if (random.nextInt(100) == 0) {
      trackClickExact(shortKey);  // 1 in 100
  }
  // Extrapolate: 1 million clicks reported as 1M tracking samples
  
  Result: 1% click count precision, 100x reduction in DB load
  ```
- Failover: If click tracking fails, don't block redirect (fire-and-forget)
```

#### Bottleneck 2: Cache Eviction

```
Problem: 50M URLs but cache max 10GB (only 50M × 200 bytes fits)
When new URL created: LRU evicts old link
User tries to access evicted link → miss → DB query (50ms latency)

Scenario: Old link from 2 years ago suddenly goes viral
Every click causes DB hit (not cached) = 10x traffic to replica

Solution: Tiered cache
- L1: Hot links (last 7 days, 1GB Redis, 99.9% hit rate)
- L2: Warm links (last 1 year, 10GB Redis, 95% hit rate)
- L3: Cold DB (older links, tail latency 50ms acceptable)

Configuration:
@Cacheable(cacheNames = "hotLinks", unless = "#result.createdDaysAgo > 7")
public Optional<Url> findByShortKey(String shortKey) {
    return repository.findByShortKey(shortKey);  // Only cache recent
}
```

#### Bottleneck 3: Concurrent Short Key Generation

```
Problem: Two requests hit "Create short link" simultaneously
Both get ID=100, both encode to same short key
One succeeds, one fails (unique constraint violated)

Solution: Database sequence with ordering
```java
// Use @GeneratedValue with SEQUENCE strategy
@Entity
public class Url {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "url_seq")
    @SequenceGenerator(name = "url_seq", sequenceName = "urls_id_seq", 
                       allocationSize = 100)  // Reserve 100 IDs per server
    private Long id;
    
    private String shortKey;
    // ...
}

// Each app instance reserves 100 IDs → no conflicts
// Server 1: IDs 1–100
// Server 2: IDs 101–200
// Server 3: IDs 201–300
```

---

## Real Project Usage

**Project:** Analytics platform for news aggregator (50M URLs/year)

**Challenges:**
1. **Performance:** News links are re-shortened constantly (same URL, different request) → wasted DB space
2. **Storage:** 50M URLs × 200 bytes = 10GB, but only 1GB cache → constant eviction
3. **Analytics:** Top news links get 1M+ clicks/day → click tracking overwhelms DB

**Solution Implemented:**

1. **Deduplication:** Check cache before creating new short link
   ```java
   String existing = cache.get("url:" + longUrl);
   if (existing != null) return existing;  // No new entry needed
   ```
   Result: 80% fewer DB writes

2. **Tiered Cache:** Hot links (today's news) in L1, older in L2
   ```
   L1 (1GB): Links from last 7 days
   L2 (10GB): Links from last 1 year
   DB: Historical archive
   ```
   Result: Cache hit rate improved 85% → 97%

3. **Probabilistic Analytics:** Sample 1% of clicks, extrapolate
   ```java
   if (random.nextInt(100) == 0) {  // 1% exact tracking
       analytics.track(shortKey, geoLocation);
   }
   ```
   Result: DB load reduced 50x for analytics

4. **Batch Click Updates:** Buffer 1000 clicks, update database once
   Result: DB insert reduced from 1000/sec to 1 batch/sec

**Final Metrics:**
- Latency: p99 < 50ms (was 200ms)
- Storage: Fit 50M URLs in 10GB
- Cost: Reduced DB load 40%, cache hits 97%
- Availability: 99.99% uptime (zero link failures)

---

## Interview Answer (2–3 Minutes)

> "For a URL shortener, I'd focus on two things: simplicity and read-heavy scale.
>
> **First, I clarify the problem:** We're not storing URLs like a web crawler. We're creating a mapping service where writes are ~10/sec but reads are ~100/sec. Key insight: it's read-heavy.
>
> **Scale estimation:** 1M URLs written per day → 12 writes/sec, 120 reads/sec, 10GB total storage — fits easily in one database.
>
> **Architecture:** API servers behind a load balancer, Redis cache with all URL mappings (10GB), PostgreSQL primary + read replica for durability.
>
> **Key design decisions:**
> 1. Use auto-increment ID + Base62 encoding for short keys (no collisions, ordered generation).
> 2. Cache all mappings (reads go 99% to cache, < 1ms). Cache misses fall through to DB (50ms).
> 3. Async click tracking (don't block redirects). Batch updates to avoid DB overload.
> 4. Deduplication: if URL already shortened, return cached short key.
>
> **Bottleneck:** Viral link (1M clicks in 1 second). Solution: Read-only Redis replicas, probabilistic click tracking (sample 1%, extrapolate), fire-and-forget if tracking fails.
>
> **Geo-distribution:** Deploy cache + DB in each region (US, EU, APAC) to minimize latency for users worldwide.
>
> The key trade-off: **consistency vs latency.** I guarantee short keys are never created twice (strong consistency) but accept eventual consistency on click counts."

**Follow-up likely:** "What if someone tries to claim a short key (like 'just-in')?" → Custom short keys need to check availability first, add prefix system (just-in-123 to avoid collisions).

---

## Quick Revision Card — URL Shortener

| Component | Choice | Why |
|---|---|---|
| **Key Generation** | Auto-increment ID + Base62 | No collisions, ordered, compact |
| **Cache** | Redis (all mappings) | 10GB fits in memory, 99% hit rate |
| **DB** | PostgreSQL primary + replica | ACID for durability, async replication |
| **Scaling Reads** | Read replicas + cache | 99% reads from cache, 1% from replica |
| **Scaling Writes** | Batch updates, dedup cache | Reduce write load 80% with dedup |
| **Geo-Distribution** | Region-based sharding | Minimize latency, locality principle |
| **Analytics** | Probabilistic sampling | Track 1%, extrapolate 100x throughput gain |
| **TTL** | 1 year (virtually infinite) | Links live for years, cache doesn't evict |

---

**End of Topic 24**
