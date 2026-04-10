# Topic 26: Design a Notification System (Email/SMS/Push)

## Concept
A distributed system that reliably delivers messages to users via multiple channels (email, SMS, push notifications). Challenges: channel diversity, reliability (at-least-once delivery), scale (millions of notifications/day), latency (< 5 seconds for critical alerts), and handling failures gracefully.

## Simple Explanation
Notification system is like a postal service:
- **Envelope:** Message template + user info (email/phone/device)
- **Sorting:** Route by channel (email → mail delivery, SMS → telecom, push → mobile app)
- **Reliability:** Track delivery status, retry if failed (dead letter handling)
- **Speed:** Batch processing for efficiency but prioritize urgent messages
- **Failure handling:** If mail carrier sick, use backup carrier (fallback channels)

---

## System Design: Step-by-Step

### STEP 1: Clarify Requirements

**Functional:**
- Support 3 channels: Email, SMS, Push notifications
- Send notification within 5 seconds for critical, 1 hour for non-critical
- Retry on failure (e.g., network timeout)
- Allow users to configure notification preferences (opt-in/out per channel)
- Track delivery status (sent, delivered, failed, bounced)

**Non-Functional:**
- **Scale:** 1 billion notifications per day (11.5K per second)
- **Latency:** P50 < 1 second, P99 < 5 seconds
- **Reliability:** At-least-once delivery with idempotency keys (effectively-once user outcome)
- **Availability:** 99.99% uptime (allow temporary delays not failures)

**Out of scope:**
- Rich media (image/video attachments)
- Personalization via ML
- User consent management (GDPR compliance)

### STEP 2: Estimate Scale

```
Notifications per day: 1 billion
Peak RPS: 1B / 86400 = 11.5K requests/sec

Distribution:
- Email: 60% = 6.9K/sec
- SMS: 30% = 3.5K/sec
- Push: 10% = 1.15K/sec

Average message size: 2KB (payload + metadata)
Per day throughput: 1B × 2KB = 2TB
Message queue size (buffering 5 min peak): 11.5K × 300 sec = 3.45M messages

Database: Track notifications
- Store 1 billion records per day
- Retention: 30 days = 30 billion records
- Query pattern: "Show me delivery status of all notifications sent today"
  Index: (userId, timestamp)
```

### STEP 3: High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Client Applications (app servers, scheduled jobs)            │
└────────────────┬────────────────────────────────────────────┘
                 │ POST /notification/send
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ Notification Service                                         │
│ - Validate (user exists, channel allowed)                   │
│ - Add to queue                                              │
│ - Return 200 (async confirmation)                           │
└────────────┬─────────────────────────────────────────────────┘
             │ Enqueue
             ▼
┌─────────────────────────────────────────────────────────────┐
│ Message Broker (Kafka/RabbitMQ)                            │
│ Topics:                                                      │
│ - notifications.email                                       │
│ - notifications.sms                                         │
│ - notifications.push                                        │
└──┬──────────────────┬──────────────────┬───────────────────┘
   │                  │                  │
   ▼                  ▼                  ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│Email Worker  │ │SMS Worker    │ │Push Worker   │
│(scale: 10)   │ │(scale: 5)    │ │(scale: 3)    │
└──┬───────────┘ └──┬───────────┘ └──┬───────────┘
   │               │                  │
   ▼               ▼                  ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│SendGrid API  │ │Twilio SMS    │ │Firebase FCM  │
│(external)    │ │(external)    │ │(external)    │
└──┬───────────┘ └──┬───────────┘ └──┬───────────┘
   │               │                  │
   └───────┬───────┴──────────────────┘
           │
           ▼
    ┌─────────────────┐
    │Status DB        │
    │(PostgreSQL)     │
    │- notification_id│
    │- user_id        │
    │- channel        │
    │- status         │
    │- timestamp      │
    └─────────────────┘
```

### STEP 4: Deep-Dive: Components

#### Component 1: Notification Service (API)

```java
@RestController
@RequestMapping("/api/notification")
public class NotificationController {
    
    @Autowired
    private NotificationService notificationService;
    
    @PostMapping("/send")
    public ResponseEntity<NotificationResponse> send(
            @RequestBody NotificationRequest request) {
        
        // 1. Validation
        if (!userPreferences.isChannelEnabled(request.userId, request.channel)) {
            return ResponseEntity.ok(new NotificationResponse(
                status = "SKIPPED",
                reason = "User opted out of " + request.channel
            ));
        }
        
        // 2. Deduplicate (prevent duplicate sends if retry)
        String dedupeKey = request.userId + ":" + request.eventId;
        if (cache.hasKey(dedupeKey)) {
            return ResponseEntity.ok(new NotificationResponse(
                status = "DUPLICATE",
                notificationId = cache.get(dedupeKey)
            ));
        }
        
        // 3. Create notification record
        Notification notification = Notification.builder()
            .userId(request.userId)
            .channel(request.channel)
            .title(request.title)
            .body(request.body)
            .status(PENDING)
            .createdAt(Instant.now())
            .build();
        
        Notification saved = notificationRepository.save(notification);
        
        // 4. Enqueue for processing
        notificationService.enqueue(saved);
        
        // 5. Update cache for deduplication
        cache.set(dedupeKey, saved.getId(), Duration.ofMinutes(15));
        
        return ResponseEntity.ok(new NotificationResponse(
            status = "ACCEPTED",
            notificationId = saved.getId()
        ));
    }
}
```

#### Component 2: Worker (Channel-Specific)

```java
@Component
public class EmailWorker {
    
    @Autowired
    private KafkaListener listener;
    
    @Autowired
    private SendGridClient sendGridClient;
    
    @Autowired
    private NotificationRepository notificationRepository;
    
    @KafkaListener(topics = "notifications.email")
    public void processBatch(List<Notification> batch) {
        // Process in batches for efficiency
        int successCount = 0;
        
        for (Notification notification : batch) {
            try {
                // 1. Fetch user details (email, language)
                User user = userService.getUser(notification.userId);
                
                // 2. Render template (personalization)
                String emailBody = templateEngine.render(
                    notification.getTemplate(),
                    Map.of(
                        "name", user.getName(),
                        "message", notification.getBody()
                    )
                );
                
                // 3. Send via SendGrid
                SendGridResponse response = sendGridClient.send(
                    from = "noreply@company.com",
                    to = user.getEmail(),
                    subject = notification.title,
                    body = emailBody
                );
                
                // 4. Update status
                if (response.isSuccess) {
                    notification.setStatus(SENT);
                    notification.setExternalId(response.messageId);
                    notification.setSentAt(Instant.now());
                    successCount++;
                } else {
                    notification.setStatus(FAILED);
                    notification.setErrorReason(response.errorMessage);
                }
                
                notificationRepository.save(notification);
                
            } catch (Exception e) {
                log.error("Failed to send email", e);
                notification.setStatus(FAILED);
                notification.setErrorReason(e.getMessage());
                notification.setRetryCount(notification.getRetryCount() + 1);
                
                // Re-queue for retry if count < 3
                if (notification.getRetryCount() < 3) {
                    kafkaProducer.send("notifications.email.retry", notification);
                } else {
                    notification.setStatus(FAILED_PERMANENT);
                }
                
                notificationRepository.save(notification);
            }
        }
        
        metrics.recordEmailsSent(successCount);
    }
}

// Retry logic: Exponential backoff
@Component
public class RetryAction {
    
    @Scheduled(fixedDelay = 60000)  // Check every 1 minute
    public void retryFailedNotifications() {
        List<Notification> failed = notificationRepository
            .findByStatusAndRetryCountLessThan(FAILED, 3);
        
        for (Notification n : failed) {
            int retryCount = n.getRetryCount() + 1;
            long backoffMs = 5000 * (long) Math.pow(2, retryCount);  // 5s, 10s, 20s
            
            if (Instant.now().isAfter(n.getUpdatedAt().plus(
                    Duration.ofMillis(backoffMs)))) {
                kafkaProducer.send("notifications." + n.getChannel(), n);
            }
        }
    }
}
```

#### Component 3: Status Tracking

```java
@Entity
@Table(name = "notifications", indexes = {
    @Index(name = "idx_user_ts", columnList = "user_id, created_at"),
    @Index(name = "idx_status", columnList = "status")
})
public class Notification {
    
    @Id
    private String id;
    
    @Column(name = "user_id")
    private String userId;
    
    @Column(name = "channel")
    @Enumerated(EnumType.STRING)
    private Channel channel;  // EMAIL, SMS, PUSH
    
    @Column(name = "status")
    @Enumerated(EnumType.STRING)
    private Status status;  // PENDING, SENT, DELIVERED, FAILED, BOUNCED
    
    @Column(name = "external_id")
    private String externalId;  // SendGrid message ID
    
    @Column(name = "error_reason")
    private String errorReason;  // Why it failed
    
    @Column(name = "retry_count")
    private Integer retryCount = 0;
    
    @Column(name = "created_at")
    private Instant createdAt;
    
    @Column(name = "sent_at")
    private Instant sentAt;
    
    @Column(name = "delivered_at")
    private Instant deliveredAt;
}

@RestController
public class StatusController {
    
    @GetMapping("/notification/{id}/status")
    public ResponseEntity<StatusResponse> getStatus(@PathVariable String id) {
        Notification notification = notificationRepository.findById(id);
        
        return ResponseEntity.ok(StatusResponse.builder()
            .status(notification.getStatus())  // SENT / DELIVERED / FAILED
            .sentAt(notification.getSentAt())
            .deliveredAt(notification.getDeliveredAt())
            .errorReason(notification.getErrorReason())
            .build());
    }
}
```

#### Component 4: User Preferences

```java
@Entity
public class NotificationPreferences {
    
    @Id
    private String userId;
    
    @Column(name = "email_enabled")
    private boolean emailEnabled = true;  // Default: enabled
    
    @Column(name = "sms_enabled")
    private boolean smsEnabled = false;   // Default: disabled (cost + opt-in)
    
    @Column(name = "push_enabled")
    private boolean pushEnabled = true;
    
    @Column(name = "quiet_hours_start")
    private LocalTime quietHoursStart;  // e.g., 22:00 (10 PM)
    
    @Column(name = "quiet_hours_end")
    private LocalTime quietHoursEnd;    // e.g., 08:00 (8 AM)
}

// Check preferences before sending
public boolean shouldSend(Notification notification) {
    User user = userService.getUser(notification.userId);
    NotificationPreferences prefs = preferencesRepository
        .findById(user.getId());
    
    // 1. Check channel preference
    if (!prefs.isChannelEnabled(notification.channel)) {
        return false;
    }
    
    // 2. Check quiet hours (unless critical alert)
    if (!notification.isCritical() && isQuietHours(prefs)) {
        return false;  // Defer until quiet hours end
    }
    
    // 3. Check frequency (max 5 emails per day)
    long emailsToday = notificationRepository
        .countByUserIdAndChannelAndCreatedAtAfter(
            user.getId(), CHANNEL.EMAIL, Instant.now().minus(Duration.ofDays(1)));
    
    if (emailsToday >= 5) {
        return false;  // Rate limit
    }
    
    return true;
}
```

### STEP 5: Bottlenecks & Solutions

#### Bottleneck 1: Slow External APIs

```
Problem: SendGrid/Twilio/Firebase have rate limits
→ Send 10K emails/sec, but SendGrid allows 100/sec
→ Queue backs up, notifications delayed > 1 hour

Scenario:
- Email worker calls SendGrid.send(email) → 200ms blocking I/O
- 10 workers × 5 emails each = 50 emails/sec
- But need 6900/sec → queue grows unbounded

Solution 1: Batch sending (SendGrid supports batch)

// Instead of:
for (email in emails) {
    sendGridClient.send(email);  // 1 API call per email
}

// Use batch:
BatchRequest batch = BatchRequest.create();
for (Email email : emails) {
    batch.add(email);
    if (batch.size() == 100) {
        sendGridClient.sendBatch(batch);  // 1 API call per 100 emails
        batch = BatchRequest.create();
    }
}
Result: 100x throughput improvement

Solution 2: Async non-blocking I/O

// Old: Blocking
String response = httpClient.post(sendGridEndpoint, email);
// Blocks worker thread while waiting
// Result: 10 workers, 10 concurrent requests max

// New: Non-blocking (async/reactive)
WebClient.create()
    .post()
    .uri("https://api.sendgrid.com/mail/send")
    .body(BodyInserters.fromValue(email))
    .retrieve()
    .mono()  // Async, non-blocking
    .subscribe(response -> {
        // Process response in callback
        updateStatus(notification, SENT);
    });

Result: 1 thread can handle 1000 concurrent requests via event loop
```

#### Bottleneck 2: Database Write Saturation

```
Problem: Writing 11.5K status updates/sec overloads DB
→ PostgreSQL max write throughput: 5-10K/sec (disk I/O bottleneck)

Solution: Async writes + eventual consistency

// Old: Synchronous update
notification.setStatus(SENT);
notificationRepository.save(notification);  // Blocking write

// New: Async + event
notification.setStatus(SENT);
applicationEventPublisher.publishEvent(
    new NotificationSentEvent(notification)
);

// Handle in background
@EventListener(NotificationSentEvent.class)
@Async
public void onNotificationSent(NotificationSentEvent event) {
    // Write to DB asynchronously (doesn't block worker)
    notificationRepository.save(event.getNotification());
}

Result:
- Worker doesn't wait for DB write
- Multiple writes batched to DB (1 batch insert per 100 notifications instead of 100 individual inserts)
- Throughput: 10x improvement
```

#### Bottleneck 3: Late Delivery During Peak Hours

```
Problem: Notifications arrive hours late during peak traffic
→ Kafka backlog growing faster than workers can consume
→ User doesn't get password reset code until too late

Solution: Priority queues

// Create separate topics by priority
Topics:
- notifications.email.critical (password reset, security alerts)
- notifications.email.high (order confirmation)
- notifications.email.normal (newsletter, promotions)

// Workers process in priority order
Worker 1: Consumes from critical (target: < 1 second latency)
Worker 2: Consumes from high (target: < 5 seconds)
Worker 3: Consumes from normal (target: < 1 hour)

Scaling:
- Add workers dynamically: if critical backlog > 1000, add 5 workers
- Shed load: if critical backlog > 10000, pause normal/high, process only critical

Result:
- Critical: P99 < 1 second (SLA met)
- Normal: Delayed but not lost
```

---

## Real Project Usage

**Project:** E-commerce platform with >10M users

**Challenges:**
1. Order confirmation emails often delayed due to DB saturation
2. SMS cost unexpectedly high (users not opting in, receiving spam SMS)
3. Push notifications failing silently (user frustration)

**Solution Implemented:**

1. **Architecture:** Kafka → Priority queues → Workers with async DB writes
   ```java
   // Separate topic for critical order confirmations
   @KafkaListener(topics = "orders.critical")
   public void sendOrderConfirmation(Order order) {
       // Uses dedicated workers, processes in < 2 seconds
       notificationService.send(
           userId = order.userId,
           channel = EMAIL,
           priority = CRITICAL,
           template = "order_confirmation",
           data = Map.of("orderId", order.id)
       );
   }
   ```

2. **User Preferences:** Strict SMS opt-in, better consent tracking
   ```java
   // SMS disabled by default, users must explicitly opt-in
   // Track TCPA compliance (SMS opt-in date, explicit consent)
   
   NotificationPreferences prefs = new NotificationPreferences();
   prefs.setSmsEnabled(false);  // Default
   prefs.setSmsOptInDate(null);
   
   // Only send SMS after explicit opt-in
   @PostMapping("/user/preferences/sms-opt-in")
   public void optInSms(@AuthenticationPrincipal User user) {
       prefs.setSmsEnabled(true);
       prefs.setSmsOptInDate(Instant.now());
       preferencesRepository.save(prefs);
   }
   ```

3. **Monitoring:** Track push notification success/failure rates
   ```java
   // Log Firebase delivery status
   @Scheduled(fixedDelay = 60000)
   public void checkPushDeliveryStatus() {
       // Firebase sends webhook for each push notification
       // Track: sent vs delivered vs failed
       double deliveryRate = firebaseMetrics.successCount / firebaseMetrics.sentCount;
       
       if (deliveryRate < 0.95) {
           alert("Push delivery rate degraded to " + deliveryRate);
       }
   }
   ```

**Results:**
- Order confirmation P99 latency: 2 seconds (was 30 minutes)
- SMS cost reduced 80% (opt-in actually increased engagement)
- Push success rate: 98.5% (vs 85% before monitoring)

---

## Interview Answer (2–3 Minutes)

> "I'd design a notification system with Kafka for async processing.
>
> **High-level flow:** Client calls notification API → service validates + saves to DB → enqueues to Kafka → channel-specific workers consume and send via SendGrid/Twilio/Firebase → status updated asynchronously.
>
> **Why Kafka?** Because it decouples sending from processing. If SendGrid is slow, the API still returns 200 immediately. Queue grows temporarily but processes eventually. Without it, SendGrid latency would block the API.
>
> **Worker architecture:** Separate workers per channel (email scale: 10, SMS scale: 5, push scale: 3) determined by throughput needs. Email cheapest and highest volume, SMS expensive so more conservative.
>
> **Reliability:** Three retry mechanisms. First, exponential backoff (5s, 10s, 20s) for transient failures. Dead-letter queue for persistent failures (store logs, manual investigation). Finally, deduplication via request ID to prevent duplicate sends if client retries.
>
> **Performance optimization:** Batch writes to DB asynchronously instead of synchronous per-notification. Use non-blocking HTTP client for external API calls so 1 worker can handle 1000 concurrent requests. Reduce DB writes from 11.5K/sec to ~500 batch inserts/sec.
>
> **Priority handling:** Critical notifications (password reset) get separate high-priority queue with dedicated workers. Normal emails (newsletters) can queue longer. During peak, shed load: process critical only if backlog exceeds threshold.
>
> **User preferences:** Store opt-in flags (email/SMS/push enabled), quiet hours, frequency caps. Check before sending. Especially important for SMS (regulatory compliance + cost).
>
> **Status tracking:** Database tracks status (pending, sent, delivered, failed). API exposes status endpoint. External webhooks from SendGrid/Firebase update delivery status for accurate end-to-end visibility.
>
> The key trade-off is accuracy vs latency. Perfect ordering would require strong consistency, but we accept eventual consistency (user sees 'sent' before 'delivered' webhook arrives). Much simpler and faster."

**Follow-up likely:** "What if Kafka broker fails?" → Dead letter handling + fallback to direct sends (slower but survives broker outage). Or: "How do you handle exactly-once delivery?" → Deduplication keys + idempotent workers (safe to replay messages).

---

## Quick Revision Card — Notification System

| Component | Tool | Reasoning |
|---|---|---|
| **Message Queue** | Kafka | High throughput, durability, replay capability |
| **Workers** | Spring + async | Non-blocking I/O, parallel processing |
| **External APIs** | Batch + async | Higher throughput, cost efficient |
| **DB Writes** | Event-driven async | Prevent saturation during peaks |
| **Retry** | Exponential backoff | Transient failures recover, don't waste resources |
| **Priorities** | Separate topics | Critical notifications never blocked by normal traffic |
| **Idempotency** | Request ID + dedup cache | Prevent duplicate sends from retries |
| **Monitoring** | Delivery status webhook | Track end-to-end success rates |

---

**End of Topic 26**
