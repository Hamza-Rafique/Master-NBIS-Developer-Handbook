# Chapter 14 — Caching

> **Part II — System Architecture**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 14.1 What is Caching?

**Caching** is the practice of storing the result of an expensive operation in fast, temporary storage so that future requests for the same data can be served immediately — without repeating the expensive operation.

```
┌──────────────────────────────────────────────────────────────┐
│                  What Caching Solves                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  WITHOUT cache:                                              │
│                                                              │
│  Request 1: Auth Service → DynamoDB → 8ms → return result   │
│  Request 2: Auth Service → DynamoDB → 8ms → return result   │
│  Request 3: Auth Service → DynamoDB → 8ms → return result   │
│  1,000,000 requests/day → 1M DynamoDB reads/day             │
│                                                              │
│  WITH cache:                                                 │
│                                                              │
│  Request 1: Auth Service → DynamoDB → 8ms → cache result    │
│  Request 2: Auth Service → cache → 0.5ms → return result ✅ │
│  Request 3: Auth Service → cache → 0.5ms → return result ✅ │
│  1,000,000 requests/day → ~50K DynamoDB reads/day           │
│  (95% cache hit rate)                                        │
│                                                              │
│  Benefits:                                                   │
│  ├── Faster response (0.5ms vs 8ms)                         │
│  ├── Lower database load (fewer reads)                      │
│  ├── Lower cost (fewer DynamoDB capacity units)             │
│  └── Higher throughput (serve more requests)                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 14.2 Caching in NBIS — Overview

NBIS uses caching at multiple layers for different purposes:

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS Caching Layers                             │
├────────────────────┬─────────────────────────────────────────┤
│ Layer              │ What is cached                         │
├────────────────────┼─────────────────────────────────────────┤
│ Application cache  │ OTP codes, session tokens,             │
│ (ElastiCache Redis)│ partner configs, biometric status flag │
├────────────────────┼─────────────────────────────────────────┤
│ Database cache     │ Frequently accessed identity records,  │
│ (DynamoDB DAX)     │ hot-path reads for verification        │
├────────────────────┼─────────────────────────────────────────┤
│ API Gateway cache  │ Static reference data,                 │
│                    │ system health endpoints                 │
├────────────────────┼─────────────────────────────────────────┤
│ CDN cache          │ Static assets (portal JS/CSS),         │
│ (CloudFront)       │ public documentation                   │
└────────────────────┴─────────────────────────────────────────┘
```

**Critical rule for NBIS:** Never cache personal identity data (demographics, biometrics, or authentication results). Cache only operational data (OTPs, sessions, configs) and non-sensitive metadata.

---

## 14.3 Amazon ElastiCache — Redis

**Amazon ElastiCache for Redis** is the primary caching service in NBIS. Redis is an in-memory key-value store — extremely fast (sub-millisecond), supports rich data structures, and has built-in TTL (time-to-live) for automatic expiry.

```
┌──────────────────────────────────────────────────────────────┐
│              Why Redis for NBIS                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Speed:     Sub-millisecond reads and writes                │
│  TTL:       Built-in key expiry (OTPs expire automatically) │
│  Atomic ops: INCR, SETNX — for counters and locks           │
│  Data types: String, Hash, List, Set, Sorted Set            │
│  Pub/Sub:   Real-time event broadcasting                    │
│  Persistence:Optional (RDB/AOF for durability if needed)    │
│                                                              │
│  ElastiCache adds:                                           │
│  ├── Multi-AZ replication (primary + replica)               │
│  ├── Automatic failover (replica promoted in ~30s)          │
│  ├── Encryption at rest (KMS) and in transit (TLS)         │
│  ├── VPC isolation (no public internet access)              │
│  └── CloudWatch metrics + alarms                            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 14.4 Use Case 1 — OTP Cache

The most critical Redis use case in NBIS. OTPs must be:
- Generated and stored instantly
- Expired automatically after 5 minutes
- Deleted immediately after use (no reuse)
- Counted for brute-force protection

```
┌──────────────────────────────────────────────────────────────┐
│                  OTP Cache Design                            │
└──────────────────────────────────────────────────────────────┘

KEY STRUCTURE:
  otp:{hashedUIN}

  Why hash the UIN? Even in Redis (internal), UINs should
  not be stored in plaintext.
  hashedUIN = SHA-256(UIN + salt)

VALUE STRUCTURE (Redis Hash):
  {
    "otp":      "847293",
    "attempts": "0",
    "channel":  "SMS",
    "createdAt":"2025-01-15T10:30:00Z"
  }

TTL: 300 seconds (5 minutes — auto-deleted by Redis)

OPERATIONS:

SEND OTP:
  // Check rate limit first
  sends_today = redis.INCR("otp_sends:{hashedUIN}:{date}")
  redis.EXPIRE("otp_sends:{hashedUIN}:{date}", 86400)

  if sends_today > 5:
    raise RateLimitExceeded("Max 5 OTPs per day per UIN")

  // Generate and store OTP
  otp = generate_6_digit_random()
  redis.HSET("otp:{hashedUIN}", {
    "otp": otp, "attempts": 0, "channel": "SMS"
  })
  redis.EXPIRE("otp:{hashedUIN}", 300)

  // Deliver via SNS
  sns.publish(phone_number, "Your code: " + otp)

VERIFY OTP:
  stored = redis.HGETALL("otp:{hashedUIN}")

  if not stored:
    raise OTPExpired("OTP expired or not found")

  attempts = int(stored["attempts"]) + 1

  if attempts > 3:
    redis.DEL("otp:{hashedUIN}")
    raise MaxAttemptsExceeded("Account locked")

  if stored["otp"] != provided_otp:
    redis.HSET("otp:{hashedUIN}", "attempts", attempts)
    raise OTPInvalid(f"Incorrect OTP. {3 - attempts} attempts left")

  // OTP correct — consume immediately
  redis.DEL("otp:{hashedUIN}")
  return AUTH_SUCCESS

COMPLETE OTP KEY LIFECYCLE:
  T=0:    HSET otp:abc123 {otp:"847293", attempts:0}
          EXPIRE otp:abc123 300
  T=45s:  HGETALL otp:abc123 → verify → DEL otp:abc123
  T=300s: Redis auto-expires key (if not already deleted)
```

---

## 14.5 Use Case 2 — Session Token Cache

After successful authentication, a session token is issued. The token must be validated on every subsequent request within that session.

```
┌──────────────────────────────────────────────────────────────┐
│                  Session Token Cache                         │
└──────────────────────────────────────────────────────────────┘

KEY STRUCTURE:
  session:{tokenId}

VALUE STRUCTURE (Redis Hash):
  {
    "uinHash":    "sha256:a1b2c3...",
    "partnerId":  "BNK-001",
    "role":       "RELYING_PARTY",
    "aal":        "AAL2",
    "issuedAt":   "2025-01-15T10:30:00Z",
    "expiresAt":  "2025-01-15T11:30:00Z"
  }

TTL: 3600 seconds (1 hour — matches JWT expiry)

FLOW:

On successful biometric auth:
  tokenId = generate_uuid()
  redis.HSET("session:{tokenId}", session_data)
  redis.EXPIRE("session:{tokenId}", 3600)
  return JWT containing tokenId as "jti" claim

On subsequent API call:
  tokenId = extract_jti_from_jwt(request.header)
  session = redis.HGETALL("session:{tokenId}")

  if not session:
    raise SessionExpired("Session not found or expired")

  // Session valid — proceed
  request.context.uin_hash  = session["uinHash"]
  request.context.role      = session["role"]
  request.context.partner   = session["partnerId"]

On biometric lock (citizen self-service):
  // Immediately invalidate all active sessions for this UIN
  sessions = redis.SMEMBERS("user_sessions:{uinHash}")
  for tokenId in sessions:
    redis.DEL("session:{tokenId}")
  redis.DEL("user_sessions:{uinHash}")

  // Set biometric locked flag
  redis.SETEX("bio_locked:{uinHash}", 86400, "1")

SESSION REVOCATION (admin revokes identity):
  // Same pattern — delete all sessions immediately
  redis.SMEMBERS("user_sessions:{uinHash}")
  → delete all → identity cannot be used even with valid JWT
```

---

## 14.6 Use Case 3 — Biometric Lock Flag Cache

When a citizen locks their biometric, the Auth Service must know about it instantly — without hitting DynamoDB on every authentication request.

```
┌──────────────────────────────────────────────────────────────┐
│              Biometric Lock Flag Cache                       │
└──────────────────────────────────────────────────────────────┘

KEY:   bio_locked:{uinHash}
VALUE: "1"
TTL:   None (permanent until unlocked)

FLOW:

Citizen locks biometrics (Resident Portal):
  redis.SET("bio_locked:{uinHash}", "1")
  redis.SET("identity_status:{uinHash}", "ACTIVE")
  // Note: DynamoDB record ALSO updated (source of truth)
  // Redis is the fast-path check

Auth Service receives biometric request:
  // Step 1: Check Redis FIRST (fast path — sub-ms)
  locked = redis.GET("bio_locked:{uinHash}")
  if locked:
    return AUTH_003_BIOMETRIC_LOCKED  ← returns in < 1ms

  // Step 2: Check identity status
  status = redis.GET("identity_status:{uinHash}")
  if status == "SUSPENDED":
    return AUTH_005_IDENTITY_SUSPENDED
  if status == "REVOKED":
    return AUTH_006_IDENTITY_REVOKED

  // Step 3: Only now fetch template and run match
  template = s3.get_object(template_ref)
  score = matcher.compare(probe, template)
  ...

WHY THIS MATTERS:
  Without Redis:
  Every auth request → DynamoDB → 8ms minimum
  1,000,000 auth/day → 1M DynamoDB reads

  With Redis:
  Locked/suspended/revoked: returns in < 1ms, no DynamoDB
  Active (cache hit): returns status in < 1ms
  Active (cache miss): falls through to DynamoDB

  Cache hit rate on status flag: ~99%
  (most identities are ACTIVE — one value, cached forever)
```

---

## 14.7 Use Case 4 — Partner Configuration Cache

Relying parties have configuration that is read on every authentication request (allowed auth modes, rate limits, certificate). This rarely changes but is read millions of times per day.

```
┌──────────────────────────────────────────────────────────────┐
│              Partner Configuration Cache                     │
└──────────────────────────────────────────────────────────────┘

KEY:   partner_config:{partnerId}
VALUE: (JSON string)
  {
    "partnerId":       "BNK-001",
    "partnerName":     "National Bank",
    "allowedModes":    ["BIOMETRIC", "OTP"],
    "allowedModalities":["FINGER", "FACE"],
    "eKYCAllowed":     true,
    "maxOTPPerDay":    10000,
    "certificate":     "-----BEGIN CERTIFICATE-----...",
    "status":          "ACTIVE"
  }

TTL: 3600 seconds (refresh every hour)

FLOW:

Auth Service receives request from BNK-001:

  // Cache lookup
  config_json = redis.GET("partner_config:BNK-001")

  if config_json:
    config = parse(config_json)     ← cache hit: < 1ms
  else:
    config = rds.query(             ← cache miss: ~5ms
      "SELECT * FROM partners WHERE partner_id = 'BNK-001'"
    )
    redis.SETEX(
      "partner_config:BNK-001",
      3600,
      json(config)
    )

  // Validate partner is allowed to make this request
  if requested_mode not in config.allowedModes:
    return AUTH_013_PARTNER_NOT_AUTHORIZED

CACHE INVALIDATION:
  When partner config is updated (Admin Service):
  redis.DEL("partner_config:{partnerId}")
  // Next request from this partner fetches fresh from RDS
  // and re-populates cache

  Why DEL instead of updating cache directly?
  → Simpler. Avoids stale data window.
  → Cache miss on next request is acceptable (once per update)
```

---

## 14.8 Use Case 5 — Rate Limit Counters

Redis atomic increment operations make it the perfect store for rate limiting counters — tracking how many requests a partner has made within a time window.

```
┌──────────────────────────────────────────────────────────────┐
│              Rate Limit Counter Cache                        │
└──────────────────────────────────────────────────────────────┘

KEY:   rate:{partnerId}:{endpoint}:{minute}
VALUE: integer (request count)
TTL:   120 seconds (2 minute window, auto-clean)

FLOW:

Auth Service receives request from BNK-001:

  minute  = current_minute()             // "2025011510301"
  key     = "rate:BNK-001:/auth:"+minute

  // Atomic increment (thread-safe)
  count = redis.INCR(key)

  if count == 1:
    redis.EXPIRE(key, 120)              // Set TTL on first request

  if count > 500:                       // 500 req/min limit for BNK-001
    return HTTP_429_RATE_LIMIT_EXCEEDED

  // Proceed with request

WHY REDIS FOR RATE LIMITING?
  INCR is atomic — no race condition even with
  100 concurrent Auth Service instances all
  incrementing the same counter simultaneously.

  With DynamoDB: conditional writes needed,
                 more complex, higher latency.
  With Redis INCR: one command, sub-millisecond,
                   naturally atomic.

SLIDING WINDOW (advanced):
  For finer-grained rate limiting:
  KEY: rate:{partnerId}:{endpoint}
  TYPE: Sorted Set
  MEMBER: requestId
  SCORE: timestamp (Unix ms)

  ZREMRANGEBYSCORE key 0 (now-60000)   // remove entries > 60s old
  count = ZCARD key
  if count > limit: reject
  ZADD key timestamp requestId
```

---

## 14.9 Cache Patterns — Hit, Miss, Invalidation

Understanding the three fundamental cache patterns is essential for correct implementation:

```
┌──────────────────────────────────────────────────────────────┐
│              Cache Patterns                                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PATTERN 1: Cache-Aside (Lazy Loading)                       │
│  Most common pattern in NBIS                                 │
│                                                              │
│  Read:                                                       │
│  1. Check cache                                              │
│     ├── HIT  → return cached value (fast path)              │
│     └── MISS → read from DB → store in cache → return       │
│                                                              │
│  Write:                                                      │
│  1. Write to DB (source of truth)                           │
│  2. Invalidate cache (DEL key)                              │
│     (NOT update cache — avoids stale data risk)             │
│                                                              │
│  Used for: partner configs, identity status flags           │
│                                                              │
│  ─────────────────────────────────────────────────────────  │
│                                                              │
│  PATTERN 2: Write-Through                                    │
│  Write to cache AND DB simultaneously                        │
│                                                              │
│  Write:                                                      │
│  1. Write to DB                                             │
│  2. Write to cache                                          │
│                                                              │
│  Benefit: Cache always warm after writes                    │
│  Risk: Extra write latency, cache may hold stale data       │
│        if DB write succeeds but cache write fails           │
│                                                              │
│  Not recommended for NBIS (too risky for identity data)     │
│                                                              │
│  ─────────────────────────────────────────────────────────  │
│                                                              │
│  PATTERN 3: TTL-Based Expiry                                 │
│  Cache auto-expires after defined period                    │
│                                                              │
│  Set cache with TTL:                                         │
│  redis.SETEX(key, 300, value)  ← expires in 5 minutes      │
│                                                              │
│  No explicit invalidation needed                            │
│  Data may be stale for up to TTL period                     │
│                                                              │
│  Used for: OTPs (300s), sessions (3600s),                   │
│            partner configs (3600s), rate counters (120s)    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 14.10 Cache Invalidation — The Hard Problem

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

In NBIS, stale cache data can cause serious problems:

```
┌──────────────────────────────────────────────────────────────┐
│          Cache Invalidation Scenarios in NBIS                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Scenario 1: Identity suspended (must invalidate immediately)│
│                                                              │
│  Admin suspends UIN 123456789012 at 14:00:00               │
│  Without cache invalidation:                                 │
│  14:00:05 - Bank calls auth → Redis says ACTIVE → MATCH    │
│  → Suspended citizen still authenticates ← CRITICAL BUG   │
│                                                              │
│  Correct approach:                                           │
│  Admin Service suspends in DynamoDB                         │
│  Admin Service calls: redis.SET("identity_status:{hash}",   │
│                                 "SUSPENDED")                │
│  Next auth request → Redis says SUSPENDED → rejected        │
│  → Instant effect, no stale data window                    │
│                                                              │
│  ─────────────────────────────────────────────────────────  │
│                                                              │
│  Scenario 2: Partner suspended (less critical)               │
│                                                              │
│  Partner config cached for 1 hour.                         │
│  Partner suspended → cache still says ACTIVE for up to 1hr │
│                                                              │
│  Correct approach:                                           │
│  Admin Service: redis.DEL("partner_config:{partnerId}")     │
│  + redis.SET("partner_blocked:{partnerId}", "1")            │
│  Auth Service checks blocked flag before partner config     │
│  → Instant effect                                           │
│                                                              │
│  ─────────────────────────────────────────────────────────  │
│                                                              │
│  Rule for NBIS:                                              │
│  Security-sensitive data (identity status, biometric lock,  │
│  partner block) → explicit invalidation on every change     │
│                                                              │
│  Non-security data (partner config, rate limits) →          │
│  TTL-based expiry is acceptable                             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 14.11 What NOT to Cache in NBIS

As important as knowing what to cache is knowing what never to cache:

```
┌──────────────────────────────────────────────────────────────┐
│              NEVER Cache in NBIS                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ❌ Biometric templates                                       │
│     Reason: Largest attack surface. If Redis is             │
│     compromised, attacker gets biometric data.              │
│     Always fetch from S3 + KMS at query time.               │
│                                                              │
│  ❌ Demographic data (name, DOB, address)                    │
│     Reason: PII — must not be in cache memory.              │
│     Cache the STATUS of a record, never the content.        │
│                                                              │
│  ❌ Authentication results                                    │
│     Reason: "Hamza authenticated successfully" cannot be    │
│     cached and replayed. Every auth must be fresh.          │
│                                                              │
│  ❌ eKYC response packets                                    │
│     Reason: eKYC requires fresh consent per transaction.    │
│     Caching a KYC packet = allowing consent bypass.         │
│                                                              │
│  ❌ Audit log entries                                         │
│     Reason: Audit logs must be written immediately to       │
│     immutable storage. Caching delays or loses entries.     │
│                                                              │
│  ❌ The UIN itself as a cache key (unprotected)              │
│     Reason: Always hash the UIN before using it as a key.   │
│     redis.GET("otp:123456789012") — WRONG                   │
│     redis.GET("otp:" + sha256(UIN+salt)) — CORRECT          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 14.12 ElastiCache Redis — Configuration

```
┌──────────────────────────────────────────────────────────────┐
│          ElastiCache Redis Cluster Configuration             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Cluster mode: Disabled (single shard)                      │
│  (NBIS data volume does not require sharding)               │
│                                                              │
│  Node type:  cache.r7g.large                                 │
│  Memory:     13 GB (sufficient for OTP + session data)      │
│                                                              │
│  Replication:                                                │
│  ├── 1 primary node (reads + writes)                        │
│  └── 1 replica node in different AZ (reads only + failover) │
│                                                              │
│  Multi-AZ: Yes (automatic failover to replica in ~30s)      │
│                                                              │
│  Encryption:                                                 │
│  ├── At rest: KMS (nbis-redis-key)                         │
│  └── In transit: TLS 1.3 (all connections encrypted)       │
│                                                              │
│  Auth:  AUTH token (password for Redis connection)          │
│         stored in AWS Secrets Manager                       │
│                                                              │
│  Access: VPC only (private subnet — no internet access)     │
│  Security group: only Auth Service + eKYC Service can       │
│                  connect (port 6379)                        │
│                                                              │
│  Backup: Daily snapshots to S3 (retained 7 days)           │
│  (For OTP/session data, backups are low priority —          │
│   data is ephemeral. Backups for config caches.)            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 14.13 DynamoDB Accelerator (DAX)

For the verification hot path, **DynamoDB DAX** provides microsecond-latency caching directly in front of DynamoDB — with zero application code changes.

```
┌──────────────────────────────────────────────────────────────┐
│              DynamoDB DAX for Verification                   │
└──────────────────────────────────────────────────────────────┘

WITHOUT DAX:
  Auth Service
    │
    ▼ GetItem
  DynamoDB
    │ 8–10ms
    ▼
  Identity record returned

WITH DAX:
  Auth Service
    │
    ▼ GetItem (same DynamoDB SDK call — no code change)
  DAX Cluster
    ├── Cache HIT  → 100–300 microseconds ← 30–100x faster
    └── Cache MISS → DynamoDB → 8ms → cached for next time

WHAT DAX CACHES:
  ├── GetItem results (identity record by UIN)
  ├── Query results (auth history by UIN)
  └── Scan results (NOT used in NBIS — no scans allowed)

WHAT DAX DOES NOT CACHE:
  └── PutItem, UpdateItem, DeleteItem (writes go directly to DynamoDB)
      DAX is read cache only

DAX TTL: Configurable per item (default 5 minutes)

DAX vs Redis for NBIS:
  ┌──────────────────────────────┬──────────────────────────┐
  │ DAX                          │ Redis                    │
  ├──────────────────────────────┼──────────────────────────┤
  │ DynamoDB reads only          │ Any key-value data       │
  │ Zero code change             │ Requires explicit calls  │
  │ Microsecond latency          │ Sub-millisecond          │
  │ No TTL control per use case  │ Fine-grained TTL per key │
  │ Use for: identity record     │ Use for: OTP, sessions,  │
  │          hot-path reads      │ rate counters, flags      │
  └──────────────────────────────┴──────────────────────────┘
```

---

## 14.14 Cache Warming

On system startup or after cache flush, the cache is empty — all requests are cache misses (cold start). **Cache warming** pre-loads frequently accessed data before traffic arrives.

```
┌──────────────────────────────────────────────────────────────┐
│              Cache Warming Strategy                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  On service startup (Spring Boot ApplicationReadyEvent):    │
│                                                              │
│  @EventListener(ApplicationReadyEvent.class)                │
│  public void warmCache() {                                   │
│                                                              │
│    // Warm partner configs (finite, manageable list)        │
│    List<Partner> partners = rds.findAllActivePartners();    │
│    for (Partner p : partners) {                             │
│      redis.setex("partner_config:" + p.getId(),            │
│                   3600, json(p));                           │
│    }                                                         │
│    log.info("Warmed {} partner configs", partners.size());  │
│                                                              │
│    // Do NOT warm identity records (billions of records)    │
│    // DAX handles hot records automatically                 │
│                                                              │
│    // Do NOT warm OTPs (generated on demand)                │
│  }                                                           │
│                                                              │
│  Result: First request after startup hits warm cache        │
│          No cold-start latency spike for critical paths     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 14.15 Cache Monitoring and Alarms

```
┌──────────────────────────────────────────────────────────────┐
│              Cache Monitoring Strategy                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ElastiCache CloudWatch Metrics:                            │
│                                                              │
│  CacheHits / CacheMisses → Hit Rate                         │
│  Target: > 90% hit rate for partner configs                 │
│  Target: > 99% hit rate for identity status flags           │
│  Alarm: hit rate < 80% for 5 minutes → investigate         │
│                                                              │
│  CurrConnections                                             │
│  Too many → connection pool exhaustion                      │
│  Alarm: > 500 connections → scale connection pool           │
│                                                              │
│  EngineCPUUtilization                                        │
│  Alarm: > 80% for 5 min → scale up node type               │
│                                                              │
│  DatabaseMemoryUsagePercentage                               │
│  Alarm: > 80% → eviction risk (keys being dropped)         │
│  Action: increase node size or reduce TTLs                  │
│                                                              │
│  ReplicationLag (replica behind primary)                     │
│  Alarm: > 1 second → replica may serve stale data          │
│                                                              │
│  Evictions                                                   │
│  > 0 → Redis is running out of memory, evicting keys       │
│  Alarm: > 100 evictions/min → critical, may drop OTPs      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 14.16 Complete NBIS Caching Architecture

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS Complete Caching Architecture              │
└──────────────────────────────────────────────────────────────┘

CITIZEN / RELYING PARTY REQUEST:
         │
         ▼
CloudFront (CDN)
├── Static portal assets cached at edge
└── API calls passed through (never cached)
         │
         ▼
API Gateway
├── Reference data endpoints: cached (300s TTL)
└── Auth / eKYC / enrollment: NEVER cached
         │
         ▼
Auth Service (ECS Fargate)
         │
         ├──► Redis (ElastiCache)          ← primary app cache
         │    ├── OTP: otp:{uinHash}
         │    ├── Sessions: session:{tokenId}
         │    ├── Bio lock: bio_locked:{uinHash}
         │    ├── Identity status: identity_status:{uinHash}
         │    ├── Partner config: partner_config:{partnerId}
         │    └── Rate counters: rate:{partnerId}:{min}
         │
         ├──► DAX Cluster                  ← DynamoDB cache
         │    ├── GetItem (identity record by UIN) → 200µs
         │    └── Falls through to DynamoDB on miss → 8ms
         │
         ├──► DynamoDB                     ← source of truth
         │    ├── Identity records
         │    └── Auth transaction log
         │
         └──► S3 + KMS                     ← NO CACHE
              └── Biometric templates (never cached)
                  Always fetched fresh, always decrypted fresh
```

---

## 14.17 Cache Key Design Reference

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS Redis Key Reference                        │
├──────────────────────────────┬───────────────────────────────┤
│ Key Pattern                  │ TTL      │ Value type        │
├──────────────────────────────┼───────────────────────────────┤
│ otp:{uinHash}                │ 300s     │ Hash              │
│ otp_sends:{uinHash}:{date}   │ 86400s   │ Integer (counter) │
│ session:{tokenId}            │ 3600s    │ Hash              │
│ user_sessions:{uinHash}      │ 3600s    │ Set of tokenIds   │
│ bio_locked:{uinHash}         │ No TTL   │ String "1"        │
│ identity_status:{uinHash}    │ No TTL   │ String (status)   │
│ partner_config:{partnerId}   │ 3600s    │ JSON String       │
│ partner_blocked:{partnerId}  │ No TTL   │ String "1"        │
│ rate:{partnerId}:{ep}:{min}  │ 120s     │ Integer (counter) │
│ app_config:{configKey}       │ 86400s   │ String            │
└──────────────────────────────┴───────────────────────────────┘

Key design rules:
├── Always use ":" as separator (enables key scanning)
├── Always hash UINs (SHA-256 + salt)
├── Always prefix by data type (otp:, session:, rate:)
├── Include time component for counters (:{date}, :{minute})
└── Maximum key length: 256 characters
```

---

## 14.18 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Caching Reference                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP ID Authentication module uses:                       │
│                                                              │
│  Redis caching for:                                          │
│  ├── OTP management (same pattern as our design)            │
│  ├── Authentication transaction tracking                    │
│  └── Partner certificate caching                            │
│                                                              │
│  MOSIP cache key conventions:                               │
│  ├── ida_otp_{transactionId}                                │
│  ├── ida_auth_txn_{transactionId}                           │
│  └── ida_partner_cert_{partnerId}                           │
│                                                              │
│  MOSIP does NOT cache:                                       │
│  ├── Identity records (always from ID Repository)           │
│  ├── Biometric templates (always from secure store)         │
│  └── Authentication decisions (always fresh)                │
│                                                              │
│  Our design aligns exactly with MOSIP's approach —          │
│  we cache operational data, never identity data.            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 14.19 Key Terms

| Term | Definition |
|------|-----------|
| **Cache** | Fast temporary storage for results of expensive operations |
| **Cache hit** | Requested data found in cache — fast path |
| **Cache miss** | Data not in cache — must fetch from source (DB, API) |
| **TTL** | Time-to-Live — how long a cache entry lives before auto-expiry |
| **ElastiCache** | AWS managed Redis/Memcached service |
| **Redis** | In-memory key-value store — primary NBIS cache |
| **DAX** | DynamoDB Accelerator — microsecond-latency cache for DynamoDB |
| **Cache-Aside** | Read from cache first; on miss, read DB and populate cache |
| **Write-Through** | Write to cache and DB simultaneously |
| **Cache invalidation** | Removing or updating a cache entry when source data changes |
| **Cache warming** | Pre-loading cache before traffic arrives to avoid cold-start latency |
| **Eviction** | Redis removing keys when memory is full (LRU policy) |
| **INCR** | Redis atomic increment — thread-safe counter operation |
| **SETEX** | Redis SET with TTL in one atomic command |
| **SETNX** | Redis SET if Not eXists — used for distributed locks |
| **Replication lag** | Delay between primary and replica — affects consistency |
| **Hot path** | The most frequently executed code path — needs lowest latency |
| **Cold start** | First requests after startup when cache is empty |
| **Sliding window** | Rate limiting using sorted sets to track requests in rolling time window |

---

## 14.20 Key Takeaways

- **Cache operational data, never identity data** — OTPs, sessions, status flags, and partner configs are safe to cache. Biometric templates, demographics, and auth results are never cached.
- **Always hash the UIN before using it as a cache key** — the cache key space itself must not expose citizen identifiers even inside the private network.
- **TTL is your safety net** — every cache entry that could go stale must have a TTL. If you forget to invalidate, the TTL ensures eventual consistency.
- **Security-sensitive flags need explicit invalidation** — biometric lock and identity status cannot wait for TTL. Invalidate immediately on every change.
- **Redis INCR is the right tool for rate limiting** — atomic, sub-millisecond, handles concurrent requests from many service instances safely.
- **DAX gives 30–100x speedup on DynamoDB reads with zero code change** — the AWS SDK talks to DAX exactly like DynamoDB. Enable it on the identity records table for the verification hot path.
- **Never cache authentication results or eKYC packets** — every authentication must be independently verified. Caching auth results allows session hijacking and consent bypass.
- **Monitor cache hit rate and evictions** — hit rate below 80% means the cache is not helping. Evictions mean memory is too small and OTPs or sessions are being silently dropped.

---

## 14.21 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 15 | High Availability — Multi-AZ, auto-scaling, health checks, failover |
| Chapter 16 | Disaster Recovery — RTO, RPO, backup strategies, cross-region |
| Chapter 43 | Transactions — atomic writes, optimistic locking, DynamoDB transactions |

---

*Chapter 14 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
