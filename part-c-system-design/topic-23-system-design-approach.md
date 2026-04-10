# Topic 23: System Design Approach & Methodology

## How Do You Approach a System Design Question?

### Concept
A structured 5-step methodology for designing large-scale systems: **Requirements clarification → High-level architecture → Deep-dive into components → Bottleneck identification → Trade-off justification**. Purpose: Demonstrate scalability thinking, not memorized solutions.

### Simple Explanation
Building a system design is like building a city:
- **Clarify requirements:** How many residents? What's the budget? Commercial or residential?
- **Design neighborhoods:** City center (API), suburbs (services), highways (network)
- **Plan utilities:** Water (database), electricity (caching), roads (load balancing)
- **Identify problems:** Traffic jams? Power outages? Water shortage?
- **Make trade-offs:** Wide roads (fast, expensive) or narrow roads (cheap, congested)?

### The 5-Step System Design Framework

```
┌─────────────────────────────────────────────────────────────┐
│ STEP 1: CLARIFY REQUIREMENTS                               │
│ - Functional (what features?)                              │
│ - Non-functional (scale, latency, availability)            │
│ - Ask: How many users? QPS? Data volume? Regions?         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 2: ESTIMATE SCALE                                     │
│ - Back-of-envelope math                                    │
│ - Users → QPS (queries per second)                         │
│ - Storage requirement → Database size                       │
│ - Bandwidth → Network capacity                             │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 3: HIGH-LEVEL ARCHITECTURE                            │
│ - Client → Load Balancer → API servers                    │
│ - Cache layer (Redis)                                      │
│ - Database layer (Primary + Replica + Sharding)           │
│ - Message queue for async tasks                            │
│ - Storage for files/media                                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 4: DEEP-DIVE INTO COMPONENTS                          │
│ - Database: consistency vs availability                     │
│ - Cache: eviction policy, invalidation                      │
│ - API: rate limiting, retries                              │
│ - Replication: leader-follower, quorum                      │
│ - Sharding: strategy, hot key problem                       │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 5: IDENTIFY & RESOLVE BOTTLENECKS                     │
│ - "This design won't scale — how to fix?"                  │
│ - Trade-off matrix: consistency vs latency vs cost         │
│ - Justify each architectural choice                         │
└─────────────────────────────────────────────────────────────┘
```

---

## Detailed Step-by-Step Walkthrough

### STEP 1: Clarify Requirements (5–10 minutes)

**Functional Requirements:**
- What are the core features?
- What read/write patterns exist?
- What's the data structure?

**Non-Functional Requirements (The 4 S's):**
- **Scale:** Users, QPS, storage, bandwidth
- **Speed (Latency):** p50, p95, p99 response times
- **Availability:** 99.9% vs 99.99% uptime?
- **Security:** Encryption, authentication, data privacy

**Example Clarification:**

```
Interviewer: "Design Twitter"
You: "Before I design, let me clarify:
- How many users do we expect? (1M? 100M? 1B?)
- What's the main use case? (post tweets, feed, search?)
- Latency requirements? (must be < 200ms?)
- How many tweets per second? Peak?
- Can we lose data? (yes for likes, no for tweets)
- Geo-distributed? (US only or worldwide?)"
```

### STEP 2: Estimate Scale (5–10 minutes)

Use **back-of-envelope math** to understand scope:

```
Example: Twitter-like service with 100M users

Monthly Active Users: 100M
Daily Active Users: 20M (20% of MAU)
Tweets per user per day: 5
Total tweets per day: 20M users × 5 = 100M tweets/day

QPS (Queries Per Second):
Tweets written: 100M / (24 × 3600) ≈ 1,157 tweets/sec
Tweets read: 10x write rate ≈ 11,570 tweets/sec (read:write = 10:1)

Storage:
Tweet size: ~500 bytes (ID, text, metadata, timestamp)
1 year tweets: 100M × 365 × 500 bytes ≈ 18TB
Images/videos: 10x → 180TB/year
With 3-year retention: ~540TB

Bandwidth:
Peak QPS read: 11,570 × 500 bytes ≈ 5.8MB/sec
Burst (2x peak): ~12MB/sec
OK for standard datacenter links (10Gbps)
```

**Key Metrics Extracted:**
- Write throughput: ~1K/sec
- Read throughput: ~10K/sec
- Storage: ~500TB (3-year retention)
- Bandwidth: ~12MB/sec peak

### STEP 3: High-Level Architecture (5–10 minutes)

```
                          ┌─ Cache Layer (Redis)
                          │  - Frequently accessed users/tweets
                          │  - Feed cache (top 1K tweets for each user)
                          │
Users ──→ Load Balancer ──→ API Servers (Horizontal) ──→ Database Layer
(Billions)  (multiple       (10–100 servers)            │
            replicas)       - POST /tweet               │
                           - GET /feed                  │
                           - GET /user/:id              └─ Primary (RW)
                                                           Replica (R-only)
                                                           Replica (R-only)

                      Message Queue (Kafka/RabbitMQ)
                      - Async: send notifications
                      - Heavy: generate feed rankings
                      - Archive: store tweets to cold storage
```

**Component Breakdown:**

| Component | Purpose | Tech |
|---|---|---|
| **Load Balancer** | Distribute traffic across servers | Nginx, HAProxy, AWS ELB |
| **API Servers** | Handle requests stateless | Spring Boot, Go, Node.js |
| **Cache** | Reduce DB load 100x | Redis (in-memory) |
| **Primary DB** | Authoritative data source | PostgreSQL, MySQL |
| **Read Replicas** | Scale read traffic | Async replication |
| **Message Queue** | Decouple async work | Kafka, RabbitMQ |
| **Object Storage** | Images, videos | S3, GCS |
| **Monitoring** | Observe system health | Prometheus + Grafana |

### STEP 4: Deep-Dive into Components (15–20 minutes)

**Database Choice & Consistency:**

```
Decision: Relational (PostgreSQL) vs NoSQL (DynamoDB)?

Relational (PostgreSQL):
✅ ACID transactions (tweets + likes atomic)
✅ Complex queries (find tweets by user + timestamp)
✅ Schema flexibility (add columns safely)
❌ Sharding is manual and painful
❌ Scaling writes requires application-level sharding

NoSQL (DynamoDB):
✅ Automatic sharding (horizontal scaling)
✅ High write throughput (milliseconds to GB/s)
❌ No transactions (tweets + likes separate)
❌ No joins (denormalization required)

Choice: PostgreSQL primary (strong consistency), DynamoDB for high-volume analytics
```

**Cache Strategy:**

```
Cache Invalidation Pattern:

Approach 1: Write-Through
┌──────────────────────────────────────────┐
│ 1. Client POST /tweet                   │
│ 2. Write to DB                          │
│ 3. Write to Cache (same transaction)    │
│ 4. Return 200 OK                        │
│ Pro: Cache always accurate              │
│ Con: Slower writes (2 round-trips)      │
└──────────────────────────────────────────┘

Approach 2: Write-Behind (Cache-Aside)
┌──────────────────────────────────────────┐
│ 1. Client POST /tweet                   │
│ 2. Write to DB                          │
│ 3. Invalidate cache entry (async)       │
│ 4. Return 200 OK immediately            │
│ Pro: Fast writes                        │
│ Con: Stale cache possible (eventual)    │
└──────────────────────────────────────────┘

Approach 3: Time-Based TTL
┌──────────────────────────────────────────┐
│ Cache expires after 5 minutes            │
│ Pro: Simple to implement                │
│ Con: Inconsistency window = TTL         │
└──────────────────────────────────────────┘

Choice: Write-Behind + TTL (5 min)
- Fast writes (main goal)
- Acceptable stale data (few minutes)
- Automatic cleanup on TTL
```

**Sharding Strategy:**

```
Problem: Single DB handles 1K writes/sec, but we need 10K+

Sharding by User ID (Range-Based):
┌──────────────────────────┐
│ Shard 0: UserID 0–1M     │
│ Shard 1: UserID 1M–2M    │
│ Shard 2: UserID 2M–3M    │
└──────────────────────────┘

Calculation:
shard_id = user_id % 10  // 10 shards

Pro: Simple routing
Con: Hot keys (celebrity tweets → Shard X overloaded)

Sharding by Tweet ID (Hash-Based):
┌──────────────────────────┐
│ Shard 0: Hash % 10 == 0  │
│ Shard 1: Hash % 10 == 1  │
│ ...                      │
└──────────────────────────┘

Pro: Load evenly distributed
Con: No range queries (expensive to scan)

Choice: Hash-based + dedicated shards for hot users
- Consistent hashing (add/remove shards without rehashing everything)
- Monitor celebrity users separately (burst capacity)
```

### STEP 5: Identify & Resolve Bottlenecks (10–15 minutes)

**Bottleneck 1: Feed Generation is Slow**

```
Problem: Getting a user's feed requires:
1. Find all users they follow (1000s)
2. Query tweets from each (1000 DB queries!)
3. Sort by timestamp
Total: 500ms+ per request

Solution: Pre-compute feeds with Kafka

Architecture:
1. User A follows User B
2. User B tweets "Hello"
3. Kafka event: "UserB tweeted — distribute to followers"
4. Background job: Add to each follower's feed cache
5. User A reads /feed — served from Redis (instant!)

Trade-off: Higher write latency (distribution takes 2 sec)
But: Read latency drops 500ms → 10ms
```

**Bottleneck 2: Database Replication Lag**

```
Problem: Users post tweet, immediately see stale feed

Timeline:
00:00 - Tweet written to Primary
00:01 - Async replicated to Replica
00:02 - User reads from Replica → sees stale feed

Solution:
1. Read-after-write consistency: Read from Primary after write (2% of reads → primary)
2. Replica position tracking: "Feed updated as of timestamp X"
3. Acceptable stale window: Show "updated 30 seconds ago"

Code example (Spring):
@Transactional(readOnly=false)  // Writes go to PRIMARY
public Tweet postTweet(Tweet tweet) {
    return repository.save(tweet);  // PRIMARY
}

@Transactional(readOnly=true)  // Reads normally go to REPLICA
// But if within 1 second of write, READ from PRIMARY
public List<Tweet> getFeed(String userId) {
    if (lastWriteAge < 1000ms) {
        readFromPrimary();  // Sacrifice 1% latency for consistency
    }
    return repository.findRecentTweets(userId);  // REPLICA
}
```

**Bottleneck 3: Cache Miss Storm ("Thundering Herd")**

```
Problem: Cache expires (TTL=5min), 10,000 concurrent requests hit DB simultaneously

Solution: Probabilistic TTL

// ❌ WRONG: All keys expire at exact time
cache.set("feed:user:123", data, TTL=300sec)  // All expire together

// ✅ CORRECT: Randomize expiration
long ttl = 300 + (random % 60);  // 300-360 seconds
cache.set("feed:user:123", data, TTL=ttl)  // Spread out expirations

Result: Misses are distributed over 60 seconds
DB load: 1667 misses/sec (spread) vs 10,000 misses/sec (storm)
```

---

## Real Project Usage

**Project Context:** Payment platform processing $500M/year (50K transactions/day)

**Bottleneck Encountered:**
- Peak QPS: 500 requests/sec
- Payment DB: Single Primary, 2 Replicas
- Issue: After Flash Sale event (5x traffic), read replicas fell behind by 30 minutes
- Users saw "payment processing" for 30 min but transaction already completed

**How I Applied Framework:**

1. **Clarify Requirements:**
   - Must show accurate payment status within 2 seconds
   - Can tolerate 1-minute eventual consistency for analytics

2. **Estimate Scale:**
   - Peak: 2500 QPS (5x normal)
   - Storage: 100GB/year
   - Must support 99.99% availability (max downtime: 52 min/year)

3. **High-Level Architecture:**
   - Payment API → MySQL Primary + 3 Replicas
   - Redis cache for user balance (1-min TTL)
   - Kafka for async event distribution
   - Monitoring: Replica lag alerting

4. **Deep-Dive:**
   - Problem: Replication lag during traffic spikes
   - Solution: Read-after-write consistency for balance checks
     ```java
     // After payment confirmation, read from PRIMARY
     if (user.lastPaymentTime < 1000ms) {
         readBalance(PRIMARY);  // Accurate balance
     } else {
         readBalance(REPLICA);  // Cached balance OK
     }
     ```
   - Implemented Circuit Breaker: If Primary unavailable, fallback to Replica (eventual consistency)

5. **Bottleneck Resolution:**
   - Added parallelReadFromMultipleReplicas: Read from 2 replicas, return latest
   - Result: Replication lag visibility < 500ms
   - 99.99% availability maintained throughout spike

---

## Interview Answer (2–3 Minutes)

> "When I approach a system design question, I use a 5-step framework to prevent ad-hoc decisions:
>
> **First, clarify requirements** — I ask about functional needs (features, data models) and non-functional needs (how many users, QPS, latency targets, availability SLA). Example: 'If it's Twitter, are we optimizing for feed speed or tweet upload speed?'
>
> **Second, estimate scale** — I do back-of-envelope math to understand if we're at 1K QPS or 1M QPS. This drives every architectural choice. Example: 'If 100M users tweet 5 times daily, that's 1K tweet writes/sec, 10K feed reads/sec.'
>
> **Third, I sketch a high-level architecture** — API servers behind a load balancer, cache layer for hot data, primary database with read replicas for scale, and a message queue for async tasks. I pick technologies that fit the scale constraints.
>
> **Fourth, I deep-dive into critical components** — database choice (relational vs NoSQL based on consistency needs), cache strategy (write-through vs write-behind), sharding strategy (hash vs range based on query patterns). I justify each choice with trade-offs.
>
> **Finally, I identify bottlenecks** — 'This design works until 10x traffic, then the cache layer becomes a bottleneck. Here's how I'd scale it: add Redis Cluster, implement probabilistic TTL to prevent thundering herd, add read-after-write consistency for critical reads.'
>
> The key is **not memorizing solutions** but thinking about **consistency vs latency vs availability** and making trade-offs transparently. Every design involves sacrifices — a good design justifies them."

**Follow-up likely:** "What happens if the primary database goes down?" → Circuit breaker, read from replicas (eventual consistency), automated failover to replica.

---

## Quick Revision Card — System Design Framework

| Step | Key Questions | Output |
|---|---|---|
| **1. Clarify** | What features? How many users? Latency target? Availability SLA? | Functional + non-functional requirements |
| **2. Estimate** | Users/sec → QPS? Storage needed? Bandwidth? | Scale metrics (1K, 100K, 1M QPS?) |
| **3. Architecture** | Load balancer? Cache layer? DB primary+replicas? Queue?| High-level diagram (5 components min) |
| **4. Deep-Dive** | Consistency vs availability? Sharding strategy? Cache invalidation? | Component-level decisions with trade-offs |
| **5. Bottlenecks** | What breaks at 10x traffic? How to scale? Acceptable trade-offs? | Scalability solutions + monitoring |

**Common Pitfalls to Avoid:**
- ❌ Diving into code immediately (should be step 4)
- ❌ Not clarifying requirements (leads to wrong design)
- ❌ Ignoring backup/failure scenarios (no SLA achieved)
- ❌ Proposing unlimited shards/replicas (cost explodes)
- ❌ Not discussing monitoring (can't tell if design works)

---

**End of Topic 23**
