# Topic 27: Design Distributed Cache with Search (1M Transactions/Day)

## Concept
A distributed cache layer that stores frequently accessed data (hot data) with efficient search/retrieval. Challenges: invalidation consistency, memory efficiency at scale, cache coherency across regions, and search queries on cached data.

## Simple Explanation
Distributed cache is like a public library's reserve desk:
- **Hot books:** Popular bestsellers kept at desk (instant access) vs. archive storage
- **Search:** Catalog (index) tells you book location + availability instantly
- **Consistency:** When book returned, mark it available on all branch computers (invalidation)
- **Scaling:** Multiple branch desks handle requests independently, but sync status periodically
- **Memory:** Reserve desk has limited space, so keep only top 20% of books (LRU eviction)

---

## System Design: Step-by-Step

### STEP 1: Clarify Requirements

**Functional:**
- Cache frequently accessed data (objects, user profiles, product data)
- Support search queries on cached data (e.g., "find products with price < $50")
- Invalidate cache on data mutations
- Multi-region deployment (US, EU, APAC with eventual consistency)

**Non-Functional:**
- **Scale:** 1M transactions/day = 11.6 requests/sec, but bursty (peak: 100K/sec during flash sales)
- **Latency:** P50 < 100ms, P99 < 500ms for cache hits
- **Hit ratio:** 80%+ (reduce database load 5x)
- **Memory footprint:** Cache working set fits in 256GB across cluster (cost optimization)
- **Consistency:** Eventual (cache lag < 5 minutes acceptable)

### STEP 2: Estimate Scale

```
Transactions per day: 1M
Read ratio: 80%, Write ratio: 20%
Reads: 800K, Writes: 200K

Estimate working set (80/20 rule):
- 80% of reads hit 20% of data
- If 10M products in database, hot set = 2M products

Memory per product: 1KB (metadata + price + description)
Hot set memory: 2M × 1KB = 2GB
With redundancy (2x): 4GB per cache node
Cache cluster: 256GB / 4GB = 64 nodes

Peak throughput: 100K req/sec
Cache node throughput: 1000 req/sec (conservative)
Nodes needed for peak: 100K / 1000 = 100 nodes

Replication factor: 3 (for HA)
Total nodes: 100 × 3 = 300 nodes (expensive! 
→ Optimization: Scale down with compression, intelligent eviction)
```

### STEP 3: High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Client Applications (read queries, writes)                   │
└────────────────┬────────────────────────────────────────────┘
                 │
    ┌────────────┴─────────────────┐
    │                              │
    ▼                              ▼
┌──────────────┐            ┌──────────────┐
│Read Path     │            │Write Path    │
│(cache first) │            │(write-through│
└──┬───────────┘            │or behind)    │
   │                         └──┬───────────┘
   ▼                            │
┌──────────────────────────────────────────────────────────────┐
│ Hash Ring Routing (Consistent Hashing)                       │
│ key = hash(product_id) → assigned to node                    │
└──┬──────────────────────────────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────────────────────────────────┐
│ Distributed Cache Layer (Redis Cluster / Memcached)         │
│                                                              │
│ Node 1 (primary)  ↔ Node 2 (replica) ↔ Node 3 (replica)    │
│ Node 4 (primary)  ↔ Node 5 (replica) ↔ Node 6 (replica)    │
│ ...                                                          │
└──┬──────────────────────────────────────────────────────────┘
   │
   ├─ Cache miss → Query database
   │
   ▼
┌──────────────────────────────────────────────────────────────┐
│ Database (PostgreSQL with indexes)                           │
│ - product index: (category, price)                           │
│ - user index: (status, last_login)                           │
└──────────────────────────────────────────────────────────────┘
```

### STEP 4: Deep-Dive: Components

#### Component 1: Routing & Consistent Hashing

```java
@Component
public class CacheRouter {
    
    private final TreeMap<Long, CacheNode> ring = new TreeMap<>();
    private final List<CacheNode> nodes;
    
    public CacheRouter(List<CacheNode> cacheNodes) {
        this.nodes = cacheNodes;
        buildRing();
    }
    
    private void buildRing() {
        // Virtual nodes for better distribution (160 per node)
        for (CacheNode node : nodes) {
            for (int i = 0; i < 160; i++) {
                long hash = MurmurHash.hash64(node.id + ":" + i);
                ring.put(hash, node);
            }
        }
    }
    
    public CacheNode getNode(String key) {
        long hash = MurmurHash.hash64(key);
        
        // Find first node >= hash (clockwise on ring)
        Long nodeHash = ring.ceilingKey(hash);
        if (nodeHash == null) {
            nodeHash = ring.firstKey();  // Wrap around
        }
        
        return ring.get(nodeHash);
    }
    
    public List<CacheNode> getReplicaNodes(String key, int replicationFactor) {
        // Get primary + replicas
        CacheNode primary = getNode(key);
        List<CacheNode> replicas = new ArrayList<>();
        replicas.add(primary);
        
        Long hash = ring.ceilingKey(MurmurHash.hash64(key));
        Iterator<Long> it = ring.tailMap(hash, false).keySet().iterator();
        
        while (replicas.size() < replicationFactor && it.hasNext()) {
            Long nextHash = it.next();
            CacheNode replica = ring.get(nextHash);
            if (!replicas.contains(replica)) {
                replicas.add(replica);
            }
        }
        
        return replicas;
    }
}
```

#### Component 2: Write Path (Cache Invalidation)

```java
@Service
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private CacheService cacheService;
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    // ❌ WRONG: Cache-aside (potential inconsistency)
    public void updateProductWrongExample(Long productId, Product updated) {
        // Update DB
        productRepository.save(updated);
        
        // Delete cache
        cacheService.delete("product:" + productId);
        
        // Problem: If app crashes between DB write and cache delete,
        // cache has stale data forever!
    }
    
    // ✅ CORRECT: Write-through (consistency guaranteed)
    @Transactional
    public void updateProduct(Long productId, Product updated) {
        // 1. Update database first (single source of truth)
        Product saved = productRepository.save(updated);
        
        // 2. Update cache immediately
        cacheService.set("product:" + productId, saved, Duration.ofHours(1));
        
        // 3. Publish event for invalidation in other regions
        eventPublisher.publishEvent(
            new ProductUpdatedEvent(productId, saved)
        );
    }
    
    // Listener for cache invalidation from other regions
    @EventListener(ProductUpdatedEvent.class)
    @Async
    public void onProductUpdated(ProductUpdatedEvent event) {
        // Publish to Kafka for replication
        kafkaProducer.send("cache-invalidation", event);
    }
}

// Cache invalidation handling (across regions)
@Component
public class CacheInvalidationListener {
    
    @KafkaListener(topics = "cache-invalidation")
    public void invalidateCache(ProductUpdatedEvent event) {
        // Receive invalidation from another region
        // Delete from local cache
        cacheService.delete("product:" + event.productId);
        
        // Locally, next read will fetch from DB and repopulate cache
    }
}
```

#### Component 3: Search on Cached Data

```java
@Service
public class SearchService {
    
    @Autowired
    private CacheService cacheService;
    
    @Autowired
    private ProductRepository productRepository;
    
    // Search for products with filters
    public List<Product> search(SearchQuery query) {
        // query = {category: "electronics", price_range: [100, 500], page: 1, limit: 20}
        
        // ❌ WRONG: Cache full result sets per query
        // Problem: Infinite cache combinations (too much memory)
        
        // ✅ CORRECT: Cache individual entities + search on demand
        
        // Step 1: Check if full result cached (for exact query match)
        String queryKey = "search:" + query.hashCode();
        List<Long> cachedIds = cacheService.get(queryKey);
        
        if (cachedIds != null) {
            // Cache hit: fetch entities, then return
            return cachedIds.stream()
                .map(id -> cacheService.get("product:" + id))
                .filter(Objects::nonNull)
                .collect(Collectors.toList());
        }
        
        // Step 2: Query database (if cache miss)
        List<Product> products = productRepository.search(query);
        
        // Step 3: Cache the result (TTL: page results expire after 1 hour)
        if (products.size() > 0) {
            List<Long> ids = products.stream()
                .map(Product::getId)
                .collect(Collectors.toList());
            cacheService.set(queryKey, ids, Duration.ofHours(1));
        }
        
        return products;
    }
}

// For range queries (price between X and Y), use Sorted Sets in Redis
@Service
public class PriceRangeSearch {
    
    public List<Product> findByPriceRange(int minPrice, int maxPrice) {
        // Redis: sorted set where score = price
        // key = "products:by_price"
        // members = [product_1, product_2, ..., product_1000]
        // scores = [99.99, 149.99, 199.99, ...]
        
        // Range query: O(log N + M) where M = results
        Set<String> productIds = redisTemplate.opsForZSet()
            .rangeByScore("products:by_price", minPrice, maxPrice);
        
        // Fetch full objects
        return productIds.stream()
            .map(id -> cacheService.get("product:" + id))
            .collect(Collectors.toList());
    }
}
```

#### Component 4: Memory Efficiency & Eviction

```java
@Configuration
public class CacheEvictionConfig {
    
    @Bean
    public LettuceConnectionFactory cacheFactory() {
        // Redis Cluster configuration
        return new LettuceConnectionFactory(
            ClusterTopologyRefresh.adaptive(),
            LettucePoolingClientConfiguration.builder()
                .commandTimeout(Duration.ofSeconds(2))
                .build()
        );
    }
    
    @Bean
    public RedisCacheManager cacheManager(LettuceConnectionFactory factory) {
        return RedisCacheManager.create(
            RedisCacheConfiguration.defaultCacheConfig()
                // Eviction policy: LRU (least recently used)
                .computePrefixWith(cacheName -> cacheName + "::")
                .entryTtl(Duration.ofHours(1))
                .serializeValuesWith(
                    SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())
                )
                .withInitialDefaultCacheConfiguration()
                .connection(factory)
        );
    }
}

// ❌ PROBLEM: Large objects in cache → memory bloat
// Product object:
- id: 1 (8 bytes)
- name: "Super Product" (50 bytes)
- description: <10KB>
- images: [url1, url2, url3] (2KB)
- reviews: [{ rating: 5, text: "Great!" }, ...] (20 records × 1KB = 20KB)
Total: ~32KB per product

For 2M hot products: 32KB × 2M = 64GB (exceeds memory!)

// ✅ SOLUTION: Selective caching + compression

@DataCache
public class ProductCache {
    
    // Only cache essential fields for searches
    public record ProductCacheKey(
        Long id,
        String name,
        Double price,
        String category,
        Integer rating
    ) {}
    
    // Compress before storing in Redis
    public String serialize(Product p) {
        ProductCacheKey key = new ProductCacheKey(
            p.getId(), p.getName(), p.getPrice(), 
            p.getCategory(), p.getRating()
        );
        
        String json = objectMapper.writeValueAsString(key);
        byte[] compressed = Snappy.compress(json.getBytes());
        return Base64.getEncoder().encodeToString(compressed);
    }
    
    public Product deserialize(String cached) {
        byte[] decompressed = Snappy.uncompress(
            Base64.getDecoder().decode(cached)
        );
        return objectMapper.readValue(decompressed, Product.class);
    }
}

// Result: 32KB → 2KB per product (16x compression)
// Memory: 64GB → 4GB (fits in cluster!)
```

### STEP 5: Bottlenecks & Solutions

#### Bottleneck 1: Cache Stampede

```
Problem: Popular product cache expires at same time
→ All 10K users hit database simultaneously
→ Database gets 10K queries in 1 second
→ Throughput drops, response time spikes

Timeline:
T=00:00:00 - Cache entry expires (TTL = 1 hour)
T=00:00:00.001 - User 1 requests, cache miss, queries DB
T=00:00:00.002 - User 2 requests, cache miss, queries DB
T=00:00:00.005 - Users 3-10K all query DB
Result: 10K DB queries in 5ms (normal: 100 queries/sec)

Solution 1: Probabilistic early expiration (XFetch)

// Don't wait for TTL to expire
Optional<Product> get(String key) {
    Product cached = redisCache.get(key);
    
    if (cached == null) {
        return loadFromDB(key);  // Cache miss
    }
    
    long ttl = redisCache.getTtl(key);
    double probability = 1.0 - Math.exp(-ttl / 3600.0);  // Exponential decay
    
    if (Math.random() < probability) {
        // Probabilistically refresh before expiration
        CompletableFuture.runAsync(() -> {
            Product refreshed = loadFromDB(key);
            redisCache.set(key, refreshed, Duration.ofHours(1));
        });
    }
    
    return cached;  // Return stale but still serve immediately
}

Result:
- Cache refreshed proactively (users don't see cache miss)
- Probability increases as TTL approaches 0
- Smooth gradient load on DB (not spike)

Solution 2: Write-through with background refresh

// On write, always update cache + queue refresh for related searches
@Transactional
public void updateProduct(Product p) {
    productRepository.save(p);
    cacheService.set("product:" + p.id, p, Duration.ofHours(1));
    
    // Refresh searches that include this product
    kafkaProducer.send("cache-refresh", new CacheRefreshEvent(
        keys = ["search:electronics", "search:under-500"]
    ));
}
```

#### Bottleneck 2: Distributed Cache Coherency

```
Problem: Multiple regions (US, EU, APAC) have inconsistent cache
→ US cache has Product A (price $100)
→ EU cache has Product A (price $150, was just updated)
→ User in EU sees $100 from cache hit in US, confusion!

Solution: Event-driven invalidation with vector clocks

@Entity
public class Product {
    @Version  // Optimistic locking
    private Long version;
    
    @Column(name = "updated_at")
    private Instant updatedAt;
}

// Write in primary region (US)
@Transactional
public void updateProduct(Product p) {
    p.setUpdatedAt(Instant.now());
    p.setVersion(p.getVersion() + 1);
    productRepository.save(p);
    
    // Broadcast invalidation to all regions
    eventPublisher.publishEvent(
        new InvalidateCacheEvent(
            productId = p.id,
            version = p.version,
            region = "US",
            timestamp = Instant.now()
        )
    );
}

// Listen in other regions (EU, APAC)
@KafkaListener(topics = "cache-invalidation")
public void invalidateStaleCache(InvalidateCacheEvent event) {
    // Only invalidate if version is newer
    Product cached = cacheService.get("product:" + event.productId);
    
    if (cached == null || cached.version < event.version) {
        cacheService.delete("product:" + event.productId);
        log.info("Invalidated stale cache for product " + event.productId);
    }
}

Result: Eventual consistency within seconds
- Write happens in US → event sent
- Event reaches EU/APAC within 100-500ms
- Stale data served < 500ms (acceptable for most use cases)
```

#### Bottleneck 3: Thundering Herd During Failover

```
Problem: Cache node fails → 1000 requests hitting same replica node
→ Single node gets DDos'd
→ Replica overloaded, cascading failure

Solution: Request coalescing + bulkhead isolation

@Component
public class RequestCoalescer {
    
    private final ConcurrentHashMap<String, CompletableFuture<Product>>
        inflightRequests = new ConcurrentHashMap<>();
    
    public CompletableFuture<Product> get(String key) {
        // Check if request already in flight
        CompletableFuture<Product> existing = inflightRequests.get(key);
        if (existing != null) {
            return existing;  // Reuse request, don't duplicate
        }
        
        // First request: create new
        CompletableFuture<Product> future = CompletableFuture.supplyAsync(() -> {
            try {
                return loadProduct(key);  // Query DB once
            } finally {
                inflightRequests.remove(key);  // Cleanup
            }
        });
        
        inflightRequests.put(key, future);
        return future;
    }
}

// Bulkhead: Separate thread pool per service (isolation)
@Bean
public Executor cacheThreadPool() {
    return new ThreadPoolExecutor(
        10,          // core threads
        20,          // max threads
        60,          // timeout
        TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(1000)  // Queue size
    );
}

Result: Coalescing reduces DB load by 90%
- 1000 requests for same key → 1 DB query
- DB query results returned to all 1000 waiters
```

---

## Real Project Usage

**Project:** E-commerce platform (100M products, 1M concurrent users during flash sales)

**Challenges:**
1. During flash sale (popular item): 50K simultaneous reads, cache miss → stampede
2. Inventory hot data changes faster than cache TTL
3. Users in EU see stale prices (cache not invalidated fast enough)

**Solution Implemented:**

```java
// 1. Cache stampede prevention via XFetch
@Cacheable(value = "products", key = "#id")
public Product getProduct(Long id) {
    // Loaded with XFetch pattern above
    return productRepository.findById(id).orElseThrow();
}

// 2. Inventory auto-refresh (write-through with background refresh)
@Transactional
public void updateInventory(Long productId, int newStock) {
    Product p = productRepository.getOne(productId);
    p.setStock(newStock);
    p.setLastInventoryUpdate(Instant.now());
    productRepository.save(p);
    
    // Refresh cache + search indexes immediately
    cacheService.set("product:" + productId, p, Duration.ofMinutes(30));
    kafkaProducer.send("inventory-updated", 
        new InventoryUpdateEvent(productId, newStock));
}

// 3. Vector clock for consistent cache across regions
@Bean
public EventPublisher cacheInvalidationPublisher() {
    return event -> {
        // Attach region + timestamp
        event.setSourceRegion("US");
        event.setTimestamp(Instant.now());
        
        kafkaProducer.send("cache-invalidation.eu", event);
        kafkaProducer.send("cache-invalidation.apac", event);
    };
}
```

**Results:**
- Flash sale: 50K users reading same product, DB load: 1 query/sec (was 50K/sec)
- Price consistency: EU/APAC sees updates within 500ms
- Cache hit ratio: 92% (across all regions)
- Memory footprint: 4GB (products compressed, only essential fields cached)

---

## Interview Answer (2–3 Minutes)

> "For distributed cache at this scale, I'd use Redis Cluster with consistent hashing.
>
> **Architecture:** Clients use hash ring routing to map keys to cache nodes. Each key hashes to a primary node + 2 replicas for high availability.
>
> **Write path:** Write-through strategy. Update database first, then cache. Publish invalidation event to Kafka for other regions. This ensures consistency — if app crashes after cache write but before DB write, no problem; next request queries DB and repopulates cache.
>
> **Search:** Cache individual product entities, not search result sets (exponential combinations). For range queries (price between X and Y), use Redis sorted sets with price as score. O(log N) lookup.
>
> **Bottleneck 1 — Cache stampede:** When hot item cache expires, all 10K users hit DB simultaneously. Prevent with probabilistic early refresh (XFetch). Refresh cache before TTL expires with probability that increases as TTL approaches zero. Smooth load on DB.
>
> **Bottleneck 2 — Memory:** Products full objects are 30KB each. For 2M hot products, that's 60GB memory. Solution: Cache only essential fields (id, name, price, rating), compress with Snappy. Reduces from 30KB to 2KB per product.
>
> **Bottleneck 3 — Distributed inconsistency:** Multiple regions have different cache states. Use vector clocks (version numbers) + event-driven invalidation. When product updated in US, event sent to EU/APAC. Only invalidate if event version is newer than cached version.
>
> **Bottleneck 4 — Thundering herd:** During node failure, 1000 requests hit same replica. Use request coalescing: if request already in flight for same key, reuse the in-flight result. Reduces DB load 90%. Separate thread pool per service (bulkhead pattern).
>
> **Metrics:** Monitor cache hit ratio (target 85%+), eviction rate, and TTL distribution. If hit ratio drops suddenly, investigate invalidation bugs or schema changes."

**Follow-up likely:** "How do you handle partial cache invalidation?" → Use cache keys with patterns (e.g., invalidate all searches matching category 'electronics') or publish detailed invalidation events.

---

## Quick Revision Card — Distributed Cache

| Concern | Solution | Reasoning |
|---|---|---|
| **Routing** | Consistent hashing | Even distribution, minimal remapping on node failure |
| **Consistency** | Write-through + events | DB as source of truth, cache as replica |
| **Stampede** | Probabilistic refresh | Smooth load, never miss on TTL expiration |
| **Memory** | Compress + selective fields | 30KB → 2KB per object, fits in 4GB cluster |
| **Search** | Sorted sets + individual caching | O(log N) range queries, high cache hit ratio |
| **Multi-region** | Vector clocks + Kafka | Eventual consistency, resolves conflicts |
| **Failures** | Request coalescing + bulkhead | Prevents cascading failures, limits blast radius |

---

**End of Topic 27**
