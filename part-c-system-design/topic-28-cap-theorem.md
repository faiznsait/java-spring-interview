# Topic 28: CAP Theorem & Consistency Models

## Concept
**CAP Theorem (Brewer's Theorem):** In a distributed system, you can guarantee at most 2 out of 3 properties:
- **Consistency (C):** All nodes see the same data at same time
- **Availability (A):** System responds to every request (no timeouts)
- **Partition Tolerance (P):** System works despite network partitions

In practice: Network partitions are inevitable (hardware failures, latency), so you always choose P. Then trade-off between C and A.

## Simple Explanation
CAP theorem is like a restaurant during a blizzard:
- **Consistency:** All waiters agree on today's specials (if one updated, all know immediately)
- **Availability:** Restaurant always takes reservations (never says "sorry, too busy")
- **Partition tolerance:** Communication cuts out between kitchen & front desk
- **Reality:** When communication fails (partition), you can't have both consistency AND availability
  - **Consistent choice:** Send one customer away to ensure kitchen & front desk agree
  - **Available choice:** Seat customer now, hoping kitchen eventually gets order

---

## System Design: Step-by-Step

### STEP 1: When Network Partition Occurs

```
NORMAL (no partition):
Replica A: user.balance = $100 ✓
Replica B: user.balance = $100 ✓
Replica C: user.balance = $100 ✓
Client can read from any, all consistent

NETWORK PARTITION (A ↔ B, C isolated):
    ┌─────────────┐
    │ Replica A   │  → Replication stopped!
    │balance=$100 │
    └─────────────┘
         X (network failure)
    ┌─────────────┐
    │ Replica B   │
    │balance=$100 │
    └─────────────┘
    
    ┌─────────────┐
    │ Replica C   │ ← If write arrives here: balance=$110
    │balance=$110 │   But A, B still have $100
    └─────────────┘

Now: Which is "correct"? $100 or $110?
- Both are valid (all committed locally)
- But they're INCONSISTENT across the system
```

### STEP 2: Consistency Models

#### Model 1: Strong Consistency (CP = Consistency + Partition Tolerance)

```java
// ❌ TRADE-OFF: Sacrifices Availability during partition

@Service
public class AccountServiceCP {
    
    @Autowired
    private ReplicaService replicaService;
    
    @Transactional
    public boolean transfer(Account from, Account to, BigDecimal amount) {
        // Quorum write: require 2/3 replicas to acknowledge
        // before returning to client
        
        List<Replica> replicas = replicaService.getAll();
        int successCount = 0;
        int requiredQuorum = (replicas.size() / 2) + 1;  // 2 out of 3
        
        for (Replica replica : replicas) {
            try {
                replica.write(
                    key = "account:" + from.id,
                    value = "balance: " + from.balance - amount
                );
                successCount++;
            } catch (TimeoutException e) {
                log.warn("Replica " + replica.id + " timeout", e);
                // Continue trying others
            }
        }
        
        if (successCount >= requiredQuorum) {
            return true;  // Guaranteed consistency
        } else {
            throw new UnavailableException(
                "Only " + successCount + " replicas available, need " + requiredQuorum
            );
            // ❌ RESULT: Customer can't withdraw money (system unavailable)
            // ✓ RESULT: All future reads see $amount deducted (consistent)
        }
    }
}

TIMELINE (CP choice):
T=00:00:00 - Client requests transfer
T=00:00:01 - Partition occurs (replica C isolated)
T=00:00:02 - Write reaches replicas A, B (2/3 quorum)
T=00:00:05 - Timeout waiting for replica C
T=00:00:06 - App logic: Got 2/3, return success to client
T=00:00:07 - Client: Transfer completed ✓
T=00:00:08 - Replica A, B: balance updated ✓
T=00:00:30 - Network partition heals
T=00:00:31 - Replica C syncs from A, B: applies transfer (resolves!)
Result: Temporary unavailability, but never inconsistent

SQL: Quorum write in PostgreSQL
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    balance DECIMAL,
    version INT
);

CREATE PUBLICATION quorum FOR TABLE accounts;
-- Replication waits for synchronous_commit = remote_apply
ALTER SYSTEM SET synchronous_commit = remote_apply;
-- Client sees consistent data, slower writes (wait for 2/3 replicas)
```

#### Model 2: High Availability (AP = Availability + Partition Tolerance)

```java
// ❌ TRADE-OFF: Sacrifices Consistency during partition

@Service
public class AccountServiceAP {
    
    @Autowired
    private ReplicaService replicas;
    
    public boolean transfer(Account from, Account to, BigDecimal amount) {
        // Write to any replica immediately (no quorum checking)
        Replica primary = replicas.getPrimary();
        
        try {
            primary.write(
                key = "account:" + from.id,
                value = "balance: " + from.balance - amount
            );
            // Return immediately (no wait for replicas)
            return true;
        } catch (Exception e) {
            // If primary fails, write to any available replica
            Replica available = replicas.getAvailable().get(0);
            available.write(key, value);
            return true;
        }
        
        // ✓ RESULT: Customer always gets response (available)
        // ❌ RESULT: Replicas may have different balances temporarily (inconsistent)
    }
}

TIMELINE (AP choice):
T=00:00:00 - Client requests transfer
T=00:00:01 - Partition occurs (replica C isolated)
T=00:00:02 - Write sent to replica A (primary)
T=00:00:02.5 - Return success immediately ✓
T=00:00:03 - Client: Transfer completed ✓
T=00:00:05 - Replica B also receives update (replication) ✓
T=00:00:10 - Replica C (isolated) doesn't receive update yet ✗

INCONSISTENT STATE:
T=00:00:05:
Replica A: balance = $90 ✓
Replica B: balance = $90 ✓
Replica C: balance = $100 ✗ (stale)
Clients reading from C see old balance!

T=00:00:30 - Partition heals
T=00:00:31 - Replica C syncs: applies transfer, now = $90 ✓
Result: Temporary inconsistency, but never unavailable
```

### STEP 3: Consistency Models Deep-Dive

```
┌─────────────────────────────────────────────────────────────┐
│ Spectrum of Consistency Models (strong → eventual)          │
└─────────────────────────────────────────────────────────────┘

STRONG CONSISTENCY
├─ Linearizability: All operations ordered in real-time
│  Use case: Bank transfers (money can't bifurcate)
│  Cost: High latency (quorum writes)
│
├─ Serializability: Like concurrent DB transactions
│  Use case: Database ACID compliance
│  Cost: Conflict resolution overhead
│
├─ Sequential Consistency: All processes see same order
│  Use case: Replicated cache (all nodes same order even if lag)
│  Cost: Moderate latency
│
├─ Causal Consistency: If A caused B, all see A before B
│  Use case: Social media (if you post, then comment, others see post-then-comment)
│  Cost: Low latency, order guaranteed FOR causally related ops
│
EVENTUAL CONSISTENCY
└─ Eventually all replicas converge (within seconds/hours)
   Use case: DNS, CDN, social media likes
   Cost: Lowest latency, temporary inconsistency visible
```

### STEP 4: Real-World CAP Trade-Off Examples

#### Example 1: Bank (CP Choice — Consistency over Availability)

```java
// ATM Transfer: Must be consistent (money can't appear/disappear)
@Service
public class BankTransferService {
    
    // Write to primary + wait for majority replicas to acknowledge
    public boolean transferMoney(String from, String to, BigDecimal amount) {
        int totalReplicas = 3;
        int requiredAck = 2;  // Majority
        int ackCount = 0;
        
        for (Replica replica : replicas) {
            try {
                if (replica.debit(from, amount) 
                    && replica.credit(to, amount)) {
                    ackCount++;
                }
            } catch (TimeoutException e) {
                // Ignore slow replicas, try others
            }
        }
        
        if (ackCount >= requiredAck) {
            // Only confirm if majority acknowledges
            // Ensures consistency: all future reads see transfer
            auditLog.record("TRANSFER", from, to, amount, "SUCCESS");
            return true;
        } else {
            // If partition isolated too many nodes, reject transfer
            // Better unavailable than inconsistent (double-charge risk)
            auditLog.record("TRANSFER", from, to, amount, "REJECTED");
            return false;
        }
    }
}

// Result: Online banking may be unavailable during partition
// But never processes duplicate/lost transfers
```

#### Example 2: Facebook Likes (AP Choice — Availability over Consistency)

```java
// Like Counts: Can tolerate stale numbers (like count shows 100 vs actual 102)
// Important: Like action always succeeds (available)

@Service
public class LikeService {
    
    @Autowired
    private RedisCluster cache;
    
    public void likePost(Long postId, Long userId) {
        // Write to local Redis immediately (no replication wait)
        cache.increment("likes:" + postId);
        cache.add("liked:" + postId, userId);
        
        // ✓ Immediate response (available)
        // Background: replicate to other data centers
        kafkaProducer.sendAsync("likes.updates", new LikeEvent(postId, userId));
        
        // Result: Like count may be 1-2 behind for 1-2 seconds
        // But user always sees "Like successful" immediately
    }
    
    public long getLikeCount(Long postId) {
        // Read from local cache (may be slightly stale)
        Long count = cache.get("likes:" + postId);
        
        // If lag > 5 minutes (unusual), query database
        if (isStaleForTooLong(count)) {
            count = database.getLikeCount(postId);
            cache.set("likes:" + postId, count);  // Update cache
        }
        
        return count;
    }
}

Result: Sometimes users see like count that's off by 1-2
But system always responds (never "temporarily unavailable")
```

#### Example 3: E-commerce Inventory (Mixed — Context Dependent)

```
Scenario: 1000 users buy last item in stock

STRONG CONSISTENCY (CP):
// Inventory service forces quorum writes
if (inventory.canBuy(itemId, qty)) {
    inventory.decrement(itemId, qty);  // Blocks until 2/3 replicas ack
    return "ORDER_CONFIRMED";
} else {
    return "OUT_OF_STOCK";
}

Problems:
- During network partition, inventory unreachable
- 1 user gets order, 999 get "temporarily unavailable"
- Users frustrated, leave site

EVENTUAL CONSISTENCY (AP):
// Inventory service optimistically accepts orders
orders.add(order);  // Accept immediately
kafkaProducer.send("orders", order);  // Replicate async

Problems:
- 5 users buy last item (all succeeded)
- Later, only 1 can ship
- 4 users get refunds (unhappy but data recovers)
- Better than 999 users getting errors

SOLUTION: Hybrid (Time-bounded Consistency)
// Allow temporarily incorrect answers for availability
// But when user checks later, show correct status

public OrderResponse buy(Order order) {
    // Check local cache (may be stale by 1-2 seconds)
    if (inventoryCache.canBuy(order)) {
        orders.add(order);
        
        // Background: Verify with authoritative inventory
        CompletableFuture.runAsync(() -> {
            boolean actuallyAvailable = authoritative
                .inventory.canBuy(order.itemId);
            
            if (!actuallyAvailable) {
                // Oops, bought unavailable item
                // Refund automatically + alert customer
                refund(order);
                notifyCustomer(order, "Item became unavailable");
            }
        });
        
        return OrderResponse.TENTATIVE_CONFIRM;
    }
}

// After 10 seconds, check confirmation
if (order.isPacked()) {
    return OrderResponse.CONFIRMED;
} else if (order.isRefunded()) {
    return OrderResponse.REFUNDED;
}
```

### STEP 5: Choosing C vs A

```
┌─────────────────────────────────────────────────────────────┐
│ CAP Theorem Decision Tree                                   │
└─────────────────────────────────────────────────────────────┘

1. Is partition likely?
   YES → Must choose C or A
   NO → Keep all 3 (single data center)

2. If partition occurs, must system respond to requests?
   YES (AP) → Accept stale data temporarily
      Examples: Likes, comments, view counts
      Tradeoff: Eventual consistency (resolves in minutes)
      
   NO (CP) → Reject unavailable requests
      Examples: Bank transfers, payment processing
      Tradeoff: Partial unavailability during partition

3. Hybrid approach: Different consistency for different operations
   
   READS:  Use cache (eventually consistent) for speed
   WRITES: Use quorum (consistent) for correctness
   
   Example: Read user profile (AP), process payment (CP)

4. Detection & Recovery
   
   How do you detect partition?
   - Network timeout thresholds
   - Failed replication in N consecutive heartbeats
   
   How do you handle stale reads during partition?
   - Accept as-is (user sees old version)
   - Serve "unknown" response (client retries)
   - Use version vectors to detect staleness
```

---

## Real Project Usage

**Project:** Fintech payment platform + real-time analytics

**Design Decisions:**

```java
// 1. PAYMENTS (CP Choice — Consistency critical)
@Service
public class PaymentServiceCP {
    
    // Quorum: 2/3 replicas must acknowledge before confirming payment
    public PaymentResult processPayment(Payment payment) {
        int quorumSize = 2;
        int successCount = 0;
        
        for (ReplicaService replica : replicas) {
            if (replica.reserveFunds(
                account = payment.fromAccount,
                amount = payment.amount)) {
                successCount++;
            }
        }
        
        if (successCount >= quorumSize) {
            // Confirm with quorum (consistent)
            return PaymentResult.SUCCESS;
        } else {
            // Reject if can't get quorum (unavailable during partition)
            return PaymentResult.UNAVAILABLE;
        }
    }
}

// 2. ANALYTICS (AP Choice — Availability critical)
@Service
public class AnalyticsServiceAP {
    
    // Write to local cache immediately
    public void recordEvent(AnalyticsEvent event) {
        analyticsCache.increment("event:" + event.type);
        
        // Replicate to data warehouse eventually
        // (no blocking)
        kafkaProducer.sendAsync("analytics", event);
    }
    
    public long getEventCount(String type) {
        // Read from cache (may lag by 1-2 seconds)
        return analyticsCache.get("event:" + type);
    }
}

// 3. INVENTORY (Hybrid — Context dependent)
@Service
public class InventoryServiceHybrid {
    
    public PurchaseResponse buy(Order order) {
        // Optimistic local cache (AP)
        if (inventoryCache.canBuy(order.itemId)) {
            orders.add(order);
            
            // Async verify with consistency checker (eventually CP)
            CompletableFuture.runAsync(() -> {
                verifyAndReserve(order);
            });
            
            return PurchaseResponse.CONFIRMED;
        }
        return PurchaseResponse.OUT_OF_STOCK;
    }
}
```

**Results:**
- Payments: 99.99% availability (accepted payments only during quorum agreement)
- Analytics: 99.999% availability (always can track events)
- Hybrid approach gave best of both (except rare 1-2 refunds on inventory)

---

## Interview Answer (2–3 Minutes)

> "CAP theorem says in a distributed system, you can guarantee at most 2 of 3: Consistency, Availability, and Partition Tolerance.
>
> **In practice, partitions are inevitable** (network failures, hardware issues), so you always have P. The real choice is C vs A.
>
> **Consistency (CP):** All nodes see same data. To guarantee during partition, use quorum writes — require majority of replicas to acknowledge before returning to client. Trade-off: If partition isolates too many replicas, you can't reach quorum and system becomes unavailable. Example: Bank transfers — better to reject than process duplicate transfers.
>
> **Availability (AP):** System responds to every request. Write to any available replica immediately, replicate asynchronously. Trade-off: During partition, different replicas have different data. Eventually consistent — resolves when partition heals. Example: Like counts on Facebook — lag 1-2 seconds is acceptable.
>
> **Real-world:** Don't choose globally. Use **hybrid approach** based on operation:
>
> - Payments: CP (quorum writes, reject during partition)
> - Analytics: AP (write to cache async, always respond)
> - Inventory: Mixed (optimistic local cache, async verify)
>
> **Detection:** Monitor network latency. If replica heartbeat fails for 5 consecutive periods (each 1 second), assume partition.
>
> **Recovery:** When partition heals, use version vectors to determine which replica is newer. Replicas with lower versions catch up (anti-entropy repair).
>
> The key trade-off isn't theoretical — it's about understanding your SLAs and user tolerance. Payments need consistency (users never want to lose money), but social media can handle stale likes for 1-2 seconds."

**Follow-up likely:** "What if you want both consistency AND availability?" → Impossible during partition per CAP theorem. But if partition unlikely (single datacenter), you can achieve both. Or, accept temporary stale reads using read replicas with eventual catch-up.

---

## Quick Revision Card — CAP Theorem

| Property | Choice | Compromise | Example |
|---|---|---|---|
| **Consistency** | CP | Availability | Bank transfers |
| **Availability** | AP | Consistency | Social media likes |
| **Hybrid** | Mixed per operation | Case-by-case | E-commerce orders |
| **Quorum** | 2/3 replicas ack | Higher latency | Payment processing |
| **Async replication** | Background sync | Temporary stale data | Analytics, caching |
| **Detection** | Failed heartbeats | False positives possible | Timeout thresholds |
| **Resolution** | Version vectors | Merge conflicts | Anti-entropy repair |

---

**End of Topic 28**
