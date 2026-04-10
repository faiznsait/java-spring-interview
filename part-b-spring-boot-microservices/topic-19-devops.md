# Topic 19: Cloud, DevOps, and Engineering Practices

## Q136. Data at Rest, in Motion, in Use

| State | What It Is | How to Protect |
|---|---|---|
| **At Rest** | Stored data — DB, S3, disk | Encryption: AES-256, database TDE, disk encryption |
| **In Motion** | Data traveling — HTTP, Kafka, DB connections | TLS 1.3, HTTPS, SSL/TLS all connections |
| **In Use** | Active in memory/CPU | Memory encryption, secure enclaves, minimize retention |

```java
// Data at rest: JPA field encryption
@Entity
public class PaymentCard {
    @Convert(converter = EncryptedStringConverter.class)
    private String cardNumber;  // Encrypted in DB, plain in application
    
    private String last4Digits;  // Store separately for display (non-sensitive)
}

// Data in motion: Enforce TLS
server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_PASSWORD}

// Kafka TLS
spring.kafka.properties.security.protocol: SSL
spring.kafka.ssl.trust-store-location: classpath:kafka.truststore.jks

// Data in use: Don't log sensitive fields
@JsonIgnore  // Excluded from serialization/logging
private String password;

@ToString.Exclude  // Excluded from Lombok toString
private String ssn;
```

---

## Q137. On-Prem vs Cloud

| Aspect | On-Premises | Cloud (AWS/GCP/Azure) |
|---|---|---|
| **Cost model** | CapEx (upfront hardware) | OpEx (pay-as-you-go) |
| **Scaling** | Manual, slow (buy hardware) | Auto-scaling, minutes |
| **Maintenance** | Your team manages hardware/OS | Provider manages underlying infra |
| **Compliance** | Full control of data | Compliance certifications available (SOC2, PCI, GDPR) |
| **Latency** | Lower for on-prem users | Variable (geographic regions available) |
| **Disaster Recovery** | Expensive (secondary DC) | Multi-AZ/region automatic, cheap |
| **Time to market** | Slow (provision weeks) | Fast (provision seconds) |

```
When On-Prem preferred:
- Strict data sovereignty (government, banking in some countries)
- Existing large hardware investment
- Predictable, stable workload (cloud becomes expensive)
- Low latency to physical network required

When Cloud preferred:
- Variable/unpredictable traffic (auto-scale)
- Fast iteration / startup
- Global users (regions)
- Avoid infrastructure operations cost
```

---

## Q138. Git Merge vs Rebase

### Visual Difference

```
Feature branch diverged from main:

Before:
main:    A -- B -- C
                \
feature:         D -- E

After MERGE:
main:    A -- B -- C -- M    (M = merge commit)
                \       /
feature:         D -- E

After REBASE:
main:    A -- B -- C -- D' -- E'   (D, E replayed on top of C)
No merge commit — linear history
```

### Code

```bash
# Merge
git checkout main
git merge feature/my-feature
# Creates merge commit (shows branch history, non-linear)

# Rebase
git checkout feature/my-feature
git rebase main
# Replays feature commits on top of main (linear history)
git checkout main
git merge feature/my-feature --ff-only  # Fast-forward merge (no merge commit)
```

### When to Use

```
Use MERGE:
✅ Main/develop branch (preserve history of when branches merged)
✅ Long-lived branches with many contributors
✅ Want to show that branch existed
✅ Safer (never rewrites history)

Use REBASE:
✅ Feature branches before merging to keep history clean
✅ Personal branches (not shared with others)
✅ Squash noisy commits: git rebase -i HEAD~5
❌ NEVER rebase public/shared branches (rewrites history → breaks others)
```

### Squash Commits Before PR

```bash
# Squash last 5 commits into 1 clean commit
git rebase -i HEAD~5

# In editor, change 'pick' to 'squash' (or 's') for commits to combine
pick abc1234 WIP
squash def5678 fix tests
squash ghi9012 fix more tests
squash jkl3456 cleanup
pick mno7890 Add order service feature  ← Keep this message

# Result: 2 commits instead of 5 — cleaner PR history
```

---

## Q139. Maven Versioning Strategy

```xml
<!-- SNAPSHOT: In development, changes on every build -->
<version>1.2.0-SNAPSHOT</version>
<!-- SNAPSHOT deployments go to Maven snapshot repo -->
<!-- CI/CD can always pull latest snapshot -->

<!-- RELEASE: Immutable, pinned version -->
<version>1.2.0</version>
<!-- RELEASE goes to release repo, never changes -->
<!-- Semantic versioning: MAJOR.MINOR.PATCH -->
<!--   MAJOR: Breaking changes -->
<!--   MINOR: New features, backward compatible -->
<!--   PATCH: Bug fixes -->
```

### Environment Strategy

```
Development branch → 1.3.0-SNAPSHOT
Feature complete   → Release: 1.3.0
Bug fix            → 1.3.1
New features       → 1.4.0-SNAPSHOT → 1.4.0
Breaking API change → 2.0.0-SNAPSHOT → 2.0.0
```

### Maven Release Plugin

```bash
# Automated release process:
mvn release:prepare
# 1. Replaces SNAPSHOT with release version
# 2. Creates Git tag (v1.3.0)
# 3. Sets next SNAPSHOT version (1.4.0-SNAPSHOT)

mvn release:perform
# 1. Checks out tag
# 2. Builds
# 3. Deploys to release repository
```

---

## Q140. Branching Strategy & Production Rollout

### GitFlow

```
main         → Always production-ready
develop      → Integration branch
feature/*    → Feature branches (from develop)
release/*    → Release preparation (from develop → merge to main + develop)
hotfix/*     → Emergency fixes (from main → merge to main + develop)
```

### Trunk-Based Development (Modern Preferred)

```
main (trunk)     → Always deployable, CI runs on every commit
feature/*        → Short-lived (< 1 day, max 3 days)
Feature flags    → Merge often, hide unfinished code behind flags
No long-lived branches

Benefits:
- Merges daily → no integration hell
- CI/CD always on main
- Feature flags decouple deploy from release
```

### Low-Risk Deployment Techniques

#### Blue-Green Deploy
```
Blue (current production):  v1.2.0 → 100% traffic
Green (new version):        v1.3.0 → 0% traffic (deployed but inactive)

Test green → Switch load balancer → Green receives 100% traffic
Instant rollback: switch LB back to blue

Risk: Runs 2 complete environments simultaneously (2× cost)
```

#### Canary Deploy
```
v1.2.0 (stable):  90% traffic
v1.3.0 (canary):  10% traffic → monitor errors/latency
↓ metrics OK
v1.3.0: 50% traffic → monitor
↓ metrics OK
v1.3.0: 100% traffic → retire v1.2.0

Risk: Some users get new version. Rollback = redirect 100% to stable.
Good for: Gradual risk reduction, real production testing
```

#### Rolling Deploy (Kubernetes Default)
```
6 pods running v1.2.0
Replace 1 pod → 5×v1.2.0, 1×v1.3.0 (test)
Replace 2 pods → 4×v1.2.0, 2×v1.3.0
...
All 6 pods → v1.3.0

Kubernetes config:
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # Max extra pods during update
    maxUnavailable: 0  # Never reduce below desired count
```

---

## Q141. Functional vs Non-Functional Requirements & MVP

### Functional Requirements
```
What the system does — features, behaviors

Examples:
- User can create an order
- System sends confirmation email
- Admin can view sales report
- User can filter products by category
```

### Non-Functional Requirements (NFRs)
```
How the system performs

Performance:   API response < 200ms at p95 under 1000 RPS
Availability:  99.9% uptime (SLA)
Scalability:   Handle 10× traffic spike during Black Friday
Security:      OWASP Top 10 compliance, encrypted PII at rest
Maintainability: New feature deployable in < 2 hours
Observability: All requests logged with correlationId, error alerting in < 5 min
```

### MVP (Minimum Viable Product)

```
MVP = Minimum feature set that delivers core value and can be tested with real users

Framework: Must Have / Should Have / Could Have / Won't Have (MoSCoW)

E-Commerce MVP example:
Must Have:   Product listing, Cart, Checkout, Payment, Order confirmation
Should Have: Search, Filters, User accounts, Order history
Could Have:  Recommendations, Reviews, Wishlist, Loyalty points
Won't Have:  Mobile app, Multi-language, Subscription model (v1)

MVP deployed → validate with users → iterate
Don't build "should have" before "must have" is validated
```

---

## Q142. Agile Practices

### Sprint Ceremonies
```
Sprint Planning:  2 hours per sprint week (2-week sprint = 4 hours)
  - Team selects backlog items from top of prioritized backlog
  - Break into tasks, estimate in story points
  - Define sprint goal

Daily Standup: 15 minutes
  - What did I do yesterday?
  - What will I do today?
  - Any blockers?

Sprint Review: 1 hour per sprint week
  - Demo working software to stakeholders
  - Gather feedback

Sprint Retrospective: 1.5 hours per sprint
  - What went well?
  - What to improve?
  - Action items (owner + date)
```

### Handling Blockers
```
Raised in standup → Scrum Master unblocks within 24 hours
Dependency on another team → tracked in risk log
Missing requirement → spike task: 1 day to research and document options
Unclear acceptance criteria → 3 Amigos: Dev + QA + PO refine before sprint
```

---

## Q143. Code Review Best Practices

### What to Check Beyond Automated Tools

```
Automated tools handle:
✅ Code formatting (checkstyle)
✅ Static analysis (SonarQube)
✅ Dependency vulnerabilities (Snyk)
✅ Test coverage (JaCoCo)

Manual review focuses on:
```

```java
// 1. Architecture & Design
- Does this respect layer boundaries? (Controller → Service → Repo)
- Is this change in the right place?
- Is there a simpler solution?

// 2. Business Logic Correctness
- Does this match the acceptance criteria?
- Edge cases handled? (null, empty, max values, negative numbers)
- Off-by-one errors in pagination, loops?

// 3. Security
- SQL injection? (Never concatenate user input into SQL)
- Sensitive data logged? (@ToString.Exclude on password fields)
- Proper authorization check? (@PreAuthorize exists for sensitive endpoints)
- Input validated? (@Valid on controller params)

// 4. Performance
- N+1 queries? (Lazy loads in loops)
- Missing index for new query?
- Unbounded queries? (findAll() without pagination on large table)
- Holding DB transactions too long?

// 5. Error Handling
- All exceptions caught appropriately?
- Transactional rollback on exception?
- Resource cleanup in finally / try-with-resources?

// 6. Test Quality
- Tests cover the actual logic, not just happy path?
- Are mocks realistic? (Do they reflect actual behavior?)
- Test names describe what they verify?
```

---

## Q144. Kubernetes (High-Level)

### What & Why

```
Kubernetes (K8s) = container orchestration platform
Manages: deployment, scaling, self-healing, load balancing, service discovery

Without K8s: Manually start containers, manual scaling, manual restarts on crash
With K8s: Declare desired state → K8s maintains it automatically
```

### Core Concepts

```yaml
# Pod — smallest deployment unit (1+ containers)
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: order-service
    image: myregistry/order-service:1.3.0
    ports:
    - containerPort: 8080
    
# Deployment — manages Pod replicas + rolling updates
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3                  # 3 instances
  selector:
    matchLabels:
      app: order-service
  template:
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:1.3.0
        resources:
          requests:
            cpu: "250m"        # 0.25 CPU core guaranteed
            memory: "512Mi"    # 512MB RAM guaranteed
          limits:
            cpu: "1000m"       # Max 1 CPU core
            memory: "1Gi"      # Max 1GB RAM
        readinessProbe:        # Only send traffic when ready
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:         # Restart if unhealthy
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
          failureThreshold: 3  # Restart after 3 failures

# Service — stable network endpoint (load balancer over pods)
apiVersion: v1
kind: Service
spec:
  selector:
    app: order-service       # Routes to all order-service pods
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP            # Internal only (LoadBalancer for external)

# HorizontalPodAutoscaler — auto-scale based on CPU/custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale up when >70% CPU
```

### Spring Boot Readiness vs Liveness

```yaml
# Spring Boot 2.3+ supports K8s probes natively
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true

# /actuator/health/liveness  → UP means "I'm alive, don't restart me"
# /actuator/health/readiness → UP means "I'm ready to receive traffic"
# App can be live but not ready (warming up caches, DB migrations)
```

### Key Benefits for Java Microservices

```
Auto-scaling:    Traffic spike → HPA adds pods → no manual intervention
Self-healing:    Pod crashes → K8s restarts it automatically
Rolling deploys: Zero-downtime deployments (default strategy)
Service discovery: order-service → http://order-service:8080 (DNS-based)
Config management: ConfigMaps for config, Secrets for credentials
Resource limits:  Prevent one service starving others (CPU/memory limits)
```

### Interview Answer
> "Kubernetes manages containerized applications — declares desired state and K8s maintains it. In our microservices setup, each service is a Deployment with 2-3 replicas for HA. Services communicate via ClusterIP Services (K8s DNS). HPA scales pods automatically at 70% CPU.
>
> Spring Boot integration: expose /actuator/health/readiness and liveness — K8s uses readiness to decide when to send traffic (don't route to warming-up pods) and liveness to decide when to restart (crashed or deadlocked pod).
>
> Rolling updates: K8s replaces pods one at a time, zero downtime. Rollback: `kubectl rollout undo deployment/order-service` — restores previous ReplicaSet in seconds."

---

## BONUS: Popular Cloud & DevOps Concepts

### Docker — Containers vs VMs

```
VM:        Hardware → Hypervisor → Guest OS → App (heavy, slow startup, ~GBs)
Container: Hardware → OS → Docker Engine → App (lightweight, fast startup, ~MBs)

Dockerfile for Spring Boot:
# Multi-stage build — smaller final image
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY . .
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:21-jre  # Only JRE in final image (smaller)
WORKDIR /app
COPY --from=builder /app/target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### CI/CD Pipeline

```
Developer pushes to feature branch:
1. GitHub Actions triggered
2. Build (mvn package)
3. Unit + Integration tests
4. SonarQube analysis
5. Docker image build
6. Push to registry
7. Deploy to DEV environment

Merge to main:
1. All above +
2. Regression tests
3. Deploy to STAGING (auto)
4. Smoke tests
5. Deploy to PROD (manual approval gate or auto with canary)
```

### 12-Factor App (Cloud-Native Principles)

```
1.  Codebase:       One repo, many deploys (dev/staging/prod)
2.  Dependencies:   Explicitly declared (pom.xml, package.json)
3.  Config:         In environment variables, not code (DB URL, API keys)
4.  Backing services: DB, cache, queue — treat as attached resources (swappable)
5.  Build/Release/Run: Strict separation, immutable releases
6.  Processes:      Stateless processes (no in-memory session)
7.  Port binding:   Self-contained (Spring Boot embedded Tomcat — no external server)
8.  Concurrency:    Scale out (horizontal), not up (vertical)
9.  Disposability:  Fast startup, graceful shutdown (handle SIGTERM)
10. Dev/Prod parity: Keep dev, staging, prod as similar as possible
11. Logs:           Treat as event streams (stdout → ELK/Splunk)
12. Admin processes: One-off tasks as one-time processes (migrations, scripts)
```

### AWS Core Services for Java Backends

```
Compute:
  EC2:       Virtual machines (full control)
  ECS/EKS:  Docker/K8s container hosting
  Lambda:   Serverless (event-driven, pay per invocation)

Storage:
  S3:        Object storage (files, images, backups)
  EBS:       Block storage (attached to EC2, like a disk)
  EFS:       Shared file system (multiple EC2s)

Database:
  RDS:       Managed PostgreSQL/MySQL/Oracle
  Aurora:    AWS-managed PostgreSQL/MySQL (5× faster, multi-AZ)
  DynamoDB:  Managed NoSQL (serverless, auto-scale)
  ElastiCache: Managed Redis/Memcached

Messaging:
  MSK:       Managed Kafka
  SQS:       Managed queue (simple, high reliability)
  SNS:       Managed pub/sub (fan-out)

Networking:
  VPC:       Virtual private network (isolate resources)
  ALB:       Application Load Balancer (HTTP routing, SSL termination)
  Route53:   DNS
  CloudFront: CDN

Secrets:
  Secrets Manager: Managed secrets with rotation
  Parameter Store: Simple key-value config
  IAM:       Identity and access management (roles, policies)
```

### Load Balancing

```
Layer 4 (TCP): Routes by IP/port — fastest, no content inspection
Layer 7 (HTTP/HTTPS): Routes by URL, headers, cookies — smart routing

AWS ALB (L7) = Smart routing:
  /api/* → backend instances
  /admin/* → admin instances
  Feature flag header → canary instances

Health checks:
  ALB polls /health every 30s
  Unhealthy threshold: 2 → removes from pool
  Healthy threshold: 3 → adds back to pool

Spring Boot:
management.endpoint.health.show-details: always
# Ensures ALB gets detailed health for proper routing
```

### Infrastructure as Code (IaC)

```yaml
# Terraform (declare infrastructure)
resource "aws_db_instance" "orders_db" {
  allocated_storage    = 100
  engine               = "postgres"
  engine_version       = "15.4"
  instance_class       = "db.t3.medium"
  db_name              = "orders"
  username             = var.db_username
  password             = var.db_password   # From secret manager
  multi_az             = true             # Automatic failover
  backup_retention_period = 7            # 7-day backup window
  deletion_protection  = true            # Can't delete accidentally
}

# Benefits:
# Reproducible (dev/staging/prod identical infra)
# Version controlled (infra changes reviewed like code)
# Disaster recovery (recreate entire env from code)
```

### Graceful Shutdown (Spring Boot + K8s)

```yaml
# application.yml
server:
  shutdown: graceful                     # Handle in-flight requests before stopping
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s      # Wait up to 30s for requests to complete
```

```java
// K8s sends SIGTERM → Spring starts graceful shutdown:
// 1. readinessProbe returns DOWN → ALB stops sending new requests
// 2. Spring waits up to 30s for in-flight requests to complete
// 3. Kafka consumer flushes commits, DB connections closed
// 4. JVM shuts down

// For Kafka: ensure all offsets committed before shutdown
@EventListener
public void onApplicationEvent(ContextClosedEvent event) {
    log.info("Graceful shutdown: committing kafka offsets...");
    kafkaListenerEndpointRegistry.stop();
}
```

---

## Quick Revision Card — Section 14

| Topic | Key Point |
|---|---|
| **Kafka vs RabbitMQ** | Kafka: retention, replay, high throughput, multiple consumer groups. RabbitMQ: complex routing, task queues, point-to-point. |
| **Kafka Internals** | Topic → Partitions → Offset. Key → same partition (ordering). Consumer group: 1 consumer per partition. |
| **Consumer Lag** | Lag = behind producer. Fix: increase concurrency, more partitions, batch consume, scale pods. |
| **Poison Messages** | Retry with backoff → DLT. DLT consumer saves to DB, alerts. Replay after fix. |
| **Feature Flags** | Gradual rollout: 0%→5%→25%→100%. Disable = instant rollback (no redeploy). |
| **SLO vs SLA** | SLA = contract (financial penalty). SLO = internal target (more aggressive). Error budget = 100% - SLO. |
| **Rollback triggers** | 5xx > 0.5%, p99 latency > 2s, error rate > 1% for 5 min. |
| **Secrets** | AWS Secrets Manager / Parameter Store. Never in properties files or Git. |
| **Data security** | At rest: AES-256 encryption. In motion: TLS everywhere. In use: don't log sensitive fields. |
| **Git Merge vs Rebase** | Merge: preserves history (use for main). Rebase: linear history (use for feature branches, never on shared branches). |
| **Blue-Green** | Two environments, switch LB. Instant rollback, 2× cost. |
| **Canary** | Gradual % shift. Monitor metrics. Roll forward or back. |
| **12-Factor App** | Config in env vars, stateless processes, treat logs as streams, fast startup/graceful shutdown. |
| **Kubernetes** | Deployment (replicas), Service (stable endpoint), HPA (auto-scale), probes (readiness/liveness). |
| **Docker** | Multi-stage build (JDK build + JRE final). Containers: lightweight, fast, share OS kernel. |
| **CI/CD** | Push → Build → Test → Sonar → Docker image → Deploy to Dev → Staging → Prod. |

---

**End of Section 14**

---

## Additional Deep-Dive (Q136-Q144)

### Release Safety Checklist

- Feature flags guard risky changes with fast rollback.
- SLO error-budget policy determines release throttle/stop decisions.
- Deployment uses canary or blue/green for blast-radius control.
- Observability and alert ownership are defined before production rollout.

### Real Project Usage

- Successful teams treat rollback as a first-class path with rehearsed runbooks, not a last-minute improvisation.
- Code review quality improves when reviewers validate operability and failure behavior, not only functional correctness.

---

## Containerization and Deployment Deep-Dive

### Docker Fundamentals (Interview Baseline)

```dockerfile
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY . .
RUN ./mvnw -q -DskipTests package

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Interview points:
- Multi-stage builds reduce image size and attack surface.
- Pin base image versions for reproducibility.
- Keep runtime image JRE-only where possible.

### Kubernetes Core Objects

| Object | Purpose |
|---|---|
| Pod | Smallest deployable runtime unit |
| Deployment | Manages replica set + rolling updates |
| Service | Stable network endpoint for pods |
| ConfigMap / Secret | External configuration and secret injection |

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: company/order-service:1.0.0
          ports:
            - containerPort: 8080
```

### CI/CD Pipeline Stages (Expected in Senior Interviews)

1. Build + unit tests
2. Static analysis + security scan
3. Package + Docker image build
4. Deploy to lower env + integration/smoke tests
5. Progressive rollout to prod (canary/blue-green)
6. Automatic rollback on SLO breach

### Interview Drill

Question: What are must-have production safeguards?

Strong answer:
- Health probes + readiness gates.
- Progressive rollout with rollback criteria.
- Immutable artifacts and reproducible builds.
- Observability before full traffic shift.
