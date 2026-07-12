# Chapter 15 — High Availability

> **Part II — System Architecture**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 15.1 What is High Availability?

**High Availability (HA)** is the ability of a system to remain operational and accessible for the maximum possible time — even when individual components fail.

In an NBIS, availability is not a feature. It is a government obligation. Citizens cannot access hospitals, banks, or border crossings if the identity verification system is down.

```
┌──────────────────────────────────────────────────────────────┐
│              Availability — What the Numbers Mean            │
├──────────────────────┬───────────────────────────────────────┤
│ Availability         │ Downtime per year                    │
├──────────────────────┼───────────────────────────────────────┤
│ 99%                  │ 3 days 15 hours 36 minutes           │
│ 99.9%   (three 9s)   │ 8 hours 45 minutes                  │
│ 99.95%               │ 4 hours 22 minutes                  │
│ 99.99%  (four 9s)    │ 52 minutes 35 seconds               │
│ 99.999% (five 9s)    │ 5 minutes 15 seconds                │
└──────────────────────┴───────────────────────────────────────┘

NBIS SLA targets:
  Verification API:   99.99%  (52 min downtime/year max)
  eKYC API:           99.99%
  Enrollment:         99.9%   (8.75 hrs downtime/year max)
  Resident portal:    99.9%
  Admin console:      99.5%
```

---

## 15.2 The Four Pillars of High Availability

```
┌──────────────────────────────────────────────────────────────┐
│              Four Pillars of HA in NBIS                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. REDUNDANCY                                               │
│     No single point of failure.                             │
│     Every critical component has at least one duplicate.    │
│     If one fails, the other takes over.                     │
│                                                              │
│  2. HEALTH CHECKING                                          │
│     Continuous monitoring of every component.               │
│     Detect failures fast and remove unhealthy               │
│     components from rotation automatically.                 │
│                                                              │
│  3. AUTO-SCALING                                             │
│     System adjusts capacity based on load.                  │
│     Handles traffic spikes without manual intervention.     │
│     Scales down during quiet periods to save cost.          │
│                                                              │
│  4. FAST FAILOVER                                            │
│     When a component fails, traffic is redirected           │
│     to healthy components in seconds — not minutes.         │
│     No manual steps required.                               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 15.3 Availability Zones — The Foundation of HA

AWS **Availability Zones (AZs)** are physically separate data centers within a region. They have independent power, cooling, and networking. A failure in one AZ does not affect others.

**Multi-AZ deployment** means running your system across at least two AZs simultaneously — so an AZ failure takes down at most half your capacity, not all of it.

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS Multi-AZ Architecture                      │
└──────────────────────────────────────────────────────────────┘

AWS Region (e.g. ap-southeast-1 Singapore)
│
├── Availability Zone A (ap-southeast-1a)
│   ├── ECS Tasks: Auth x2, eKYC x1, Registration x1
│   ├── RDS Primary
│   ├── ElastiCache Primary
│   └── Private Subnet A (10.0.1.0/24)
│
├── Availability Zone B (ap-southeast-1b)
│   ├── ECS Tasks: Auth x2, eKYC x1, Registration x1
│   ├── RDS Standby (Multi-AZ)
│   ├── ElastiCache Replica
│   └── Private Subnet B (10.0.2.0/24)
│
└── Availability Zone C (ap-southeast-1c)
    ├── ECS Tasks: Auth x1, eKYC x1
    ├── Private Subnet C (10.0.3.0/24)
    └── (additional capacity for burst)

Internal ALB spans all 3 AZs:
→ Traffic distributed across all healthy tasks in all AZs
→ AZ-a fails → ALB routes only to AZ-b and AZ-c tasks
→ Citizens experience no interruption
```

---

## 15.4 ECS Fargate — High Availability Configuration

```
┌──────────────────────────────────────────────────────────────┐
│         ECS Service HA Configuration                         │
└──────────────────────────────────────────────────────────────┘

Auth Service (highest criticality):
{
  "serviceName": "nbis-auth-service",
  "desiredCount": 6,                    ← 6 tasks total
  "placementConstraints": [{
    "type": "spread",
    "field": "attribute:ecs.availability-zone"
  }],                                   ← spread evenly across AZs
  "deploymentConfiguration": {
    "minimumHealthyPercent": 100,       ← never drop below 6 tasks
    "maximumPercent": 150               ← max 9 tasks during deploy
  },
  "capacityProviderStrategy": [{
    "capacityProvider": "FARGATE",
    "weight": 1
  }, {
    "capacityProvider": "FARGATE_SPOT",
    "weight": 0                         ← no SPOT for critical service
  }]
}

Task distribution (6 tasks, 3 AZs):
  AZ-a: 2 tasks
  AZ-b: 2 tasks
  AZ-c: 2 tasks

AZ failure scenario:
  AZ-a goes down → 2 tasks lost
  Remaining: 4 tasks in AZ-b and AZ-c
  ALB routes traffic to remaining 4 tasks
  ECS launches 2 replacement tasks in surviving AZs
  → System runs at reduced but functional capacity
  → Auto-scaling triggers to restore full count
```

---

## 15.5 Auto-Scaling

**Auto-scaling** automatically adjusts the number of running ECS tasks based on current demand. NBIS uses three types:

### 15.5.1 Target Tracking Scaling (Primary)

```
┌──────────────────────────────────────────────────────────────┐
│              Target Tracking Auto-Scaling                    │
└──────────────────────────────────────────────────────────────┘

CONCEPT:
  Pick a metric and a target value.
  Auto-scaling adds tasks when metric exceeds target.
  Auto-scaling removes tasks when metric falls below target.
  AWS manages the scaling math automatically.

AUTH SERVICE CONFIGURATION:
  Metric:           ECSServiceAverageCPUUtilization
  Target value:     60%  (add tasks if CPU > 60%)
  Min capacity:     6    (never go below 6 tasks)
  Max capacity:     50   (never exceed 50 tasks)
  Scale out cooldown: 60s   (wait 60s before adding more)
  Scale in cooldown:  300s  (wait 5min before removing)

SCENARIO:
  Normal load: 6 tasks, CPU at 35%
  National enrollment campaign starts:
  Requests spike → CPU rises to 65%
  Auto-scaling: add 3 tasks (target 60% CPU)
  New tasks start, absorb load → CPU drops to 55%
  Campaign ends: load drops → CPU falls to 20%
  After 5 minutes: auto-scaling removes excess tasks
  Back to 6 tasks, CPU at 35%
  → All automatic, no on-call engineer needed at 2am

METRICS USED PER SERVICE:
┌────────────────────────────┬──────────────────────────────┐
│ Service                    │ Scaling Metric               │
├────────────────────────────┼──────────────────────────────┤
│ Auth Service               │ CPU utilization (60% target) │
│ eKYC Service               │ CPU utilization (60% target) │
│ Registration Service       │ SQS queue depth              │
│ Biometric Service          │ SQS queue depth              │
│ Deduplication Service      │ SQS queue depth              │
│ Notification Service       │ SQS queue depth              │
└────────────────────────────┴──────────────────────────────┘
```

### 15.5.2 Step Scaling (For SQS-Driven Services)

```
┌──────────────────────────────────────────────────────────────┐
│              Step Scaling for Queue-Driven Services          │
└──────────────────────────────────────────────────────────────┘

CONCEPT:
  Scale in steps based on queue depth.
  Different queue depths trigger different scaling actions.
  More aggressive scaling for larger backlogs.

BIOMETRIC SERVICE STEP SCALING:
  Queue: nbis-biometric-processing
  Metric: ApproximateNumberOfMessagesVisible

  Step 1: Queue depth 100–500   → add 2 tasks
  Step 2: Queue depth 500–1000  → add 5 tasks
  Step 3: Queue depth 1000–5000 → add 10 tasks
  Step 4: Queue depth > 5000    → add 20 tasks (max)

  Scale in (remove tasks):
  Queue depth < 100 for 5 min → remove 2 tasks
  Queue depth < 10  for 5 min → remove to minimum

SCENARIO:
  Normal: 2 Biometric tasks, queue depth: 5
  Morning enrollment rush:
    08:00 → 500 citizens enroll → queue depth: 200
    Step 1 fires → add 2 tasks (total: 4)
    08:15 → queue depth: 800
    Step 2 fires → add 5 tasks (total: 9)
    08:30 → queue draining → depth: 50
    Scale in → reduce to 4 tasks
    09:00 → queue empty → reduce to 2 tasks
```

### 15.5.3 Scheduled Scaling

```
┌──────────────────────────────────────────────────────────────┐
│              Scheduled Scaling for Predictable Load          │
└──────────────────────────────────────────────────────────────┘

CONCEPT:
  Pre-scale before known traffic events.
  Faster than reactive scaling (no warmup delay).

NBIS EXAMPLES:

  Government enrollment campaign (announced date):
  Day before: scale Auth to 30 tasks (up from 6)
  Campaign day: maintain high capacity
  Day after: scale back to 6

  Daily peak hours (enrollment centers 08:00-17:00):
  07:30: scale Registration to 10 tasks (up from 3)
  17:30: scale Registration back to 3 tasks

  Monthly salary day (banks call eKYC more):
  Day before month end: scale eKYC to 20 tasks (up from 6)
  2 days after: scale back to 6

CONFIGURATION:
  aws application-autoscaling put-scheduled-action \
    --service-namespace ecs \
    --resource-id service/nbis-prod/nbis-auth-service \
    --scheduled-action-name "morning-scale-up" \
    --schedule "cron(30 7 * * ? *)" \
    --scalable-target-action MinCapacity=15,MaxCapacity=50
```

---

## 15.6 RDS High Availability — Multi-AZ

```
┌──────────────────────────────────────────────────────────────┐
│              RDS Multi-AZ Architecture                       │
└──────────────────────────────────────────────────────────────┘

SETUP:
  Primary RDS instance: AZ-a (ap-southeast-1a)
  Standby RDS instance: AZ-b (ap-southeast-1b)
  Synchronous replication between primary and standby

NORMAL OPERATION:
  All reads and writes → Primary (AZ-a)
  Standby receives real-time sync copy
  Standby handles zero traffic (it is a hot spare)

  Application connects to:
  nbis-rds.cluster-xyz.ap-southeast-1.rds.amazonaws.com
  (DNS endpoint — always points to current primary)

AZ-a FAILURE SCENARIO:
  T=0:  Primary RDS in AZ-a fails
  T=30s: AWS detects failure (health check)
  T=60s: Standby in AZ-b promoted to Primary
  T=60s: DNS endpoint updated to point to new primary
  T=90s: Application reconnects (connection pool retry)
  T=90s: Full read/write restored

  Total downtime: ~60–90 seconds
  Data loss: Zero (synchronous replication)

READ REPLICAS (for reporting queries):
  Additional RDS instances in AZ-c
  Asynchronous replication from primary
  Used ONLY for read-heavy reporting (audit log queries)
  Never used for auth or enrollment (may be slightly behind)

  Audit team queries → Read Replica
  Auth Service writes → Primary only
```

---

## 15.7 DynamoDB High Availability

DynamoDB is inherently highly available — no special configuration needed, but understanding its model is important:

```
┌──────────────────────────────────────────────────────────────┐
│              DynamoDB HA Model                               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Storage:                                                    │
│  ├── Every item replicated across 3 AZs automatically       │
│  ├── A write is acknowledged only after 2/3 replicas        │
│  │   confirm storage (quorum write)                         │
│  └── Single AZ failure → zero impact, transparent          │
│                                                              │
│  Consistency models:                                         │
│  ├── Eventually consistent read (default)                   │
│  │   ├── May return data up to 1 second old                │
│  │   └── Higher throughput, lower cost                      │
│  │                                                          │
│  └── Strongly consistent read                               │
│      ├── Always returns most recent data                    │
│      └── 2x the capacity unit cost                         │
│                                                              │
│  NBIS consistency choices:                                   │
│  ├── Auth Service GetItem (identity record):                │
│  │   ConsistentRead: TRUE ← must have latest status        │
│  │   (cannot auth with stale "SUSPENDED" status)           │
│  │                                                          │
│  └── Enrollment status read:                               │
│      ConsistentRead: FALSE ← eventual ok (status display)  │
│                                                              │
│  DynamoDB Global Tables (Multi-Region):                      │
│  ├── Replicates across multiple AWS regions                 │
│  ├── Active-active: read/write in any region                │
│  └── For NBIS: consider only if deploying across           │
│      multiple countries / disaster recovery regions         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 15.8 ElastiCache High Availability

```
┌──────────────────────────────────────────────────────────────┐
│              ElastiCache Redis HA Configuration              │
└──────────────────────────────────────────────────────────────┘

SETUP:
  Primary node: AZ-a (handles reads + writes)
  Replica node: AZ-b (handles reads + failover)
  Asynchronous replication primary → replica

NORMAL OPERATION:
  Auth Service writes OTP → Primary (AZ-a)
  Auth Service reads OTP → Primary (AZ-a)
  (all traffic to primary for consistency)

PRIMARY FAILURE SCENARIO:
  T=0:   Primary Redis node in AZ-a fails
  T=30s: ElastiCache detects failure
  T=30s: Replica in AZ-b promoted to Primary
  T=30s: DNS endpoint updated automatically
  T=60s: Application reconnects to new primary
  Total downtime: ~30–60 seconds

DATA LOSS RISK:
  Async replication → replica may be slightly behind
  OTPs generated in last few seconds before failure
  may be lost on the replica.

  Acceptable for NBIS? YES for OTPs:
  ├── Citizen requests new OTP (30 seconds effort)
  ├── OTP is ephemeral by design
  └── Sessions can be re-authenticated

  NOT acceptable for: identity status flags, partner blocks
  → These are also in DynamoDB (source of truth)
  → Redis is a cache — DynamoDB is the backup

REDIS CLUSTER MODE (for larger deployments):
  Shards data across multiple primary nodes
  Each shard has its own replica
  Useful when dataset exceeds single node memory
  Not needed for standard NBIS (OTP + session data is small)
```

---

## 15.9 API Gateway High Availability

```
┌──────────────────────────────────────────────────────────────┐
│              API Gateway HA — Built In                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  AWS API Gateway is a fully managed service.                │
│  AWS guarantees 99.95% availability SLA.                    │
│  Multi-AZ by design — no configuration needed.             │
│                                                              │
│  What engineers must configure:                             │
│  ├── Throttling limits (to protect backend services)        │
│  ├── Usage plans (per relying party)                        │
│  ├── WAF rules (to block malicious traffic)                 │
│  └── CloudWatch alarms (4XX, 5XX rate monitoring)          │
│                                                              │
│  5XX errors from API Gateway indicate:                      │
│  ├── Backend service down (ECS tasks unhealthy)             │
│  ├── Integration timeout (backend too slow)                 │
│  └── Throttling (too many requests)                         │
│                                                              │
│  Alarm:                                                      │
│  5XXError rate > 1% for 5 minutes → P1 alert               │
│  4XXError rate > 5% for 5 minutes → P2 alert               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 15.10 SQS High Availability

```
┌──────────────────────────────────────────────────────────────┐
│              SQS HA — Built In                               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SQS is fully managed and inherently HA:                    │
│  ├── Messages stored across multiple AZs automatically      │
│  ├── AWS guarantees 99.9% availability                      │
│  └── No configuration needed for redundancy                │
│                                                              │
│  HA considerations for NBIS:                                │
│  ├── Message retention: 7 days                             │
│  │   If consumer is down for hours, messages wait safely   │
│  │                                                          │
│  ├── DLQ retention: 14 days                               │
│  │   Time for operations team to investigate               │
│  │                                                          │
│  └── Consumer auto-scaling                                 │
│      If one consumer AZ fails, consumers in other AZs      │
│      continue processing (SQS is not AZ-bound)             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 15.11 Complete Failover Scenarios

### Scenario 1 — Single ECS Task Failure

```
TIMELINE:
T=0s    Auth Service Task in AZ-a crashes (OOM, exception)
T=10s   ALB health check detects failure (3 checks × 10s = 30s)
T=30s   Task removed from ALB target group
T=30s   Traffic rerouted to remaining 5 healthy tasks
T=35s   ECS detects task count below desired (6)
T=60s   ECS launches replacement task in AZ-a
T=90s   New task passes health check (2 checks × 10s = 20s)
T=90s   New task added to ALB target group
T=90s   Back to 6 healthy tasks

CITIZEN IMPACT: Zero
  Requests during T=0–T=30 already in flight completed
  From T=30 onward: 5 tasks handle load (slight increase per task)
  Total: seamless, zero downtime
```

### Scenario 2 — Full AZ Failure

```
TIMELINE:
T=0s    AZ-a (ap-southeast-1a) goes down
        Loses: 2 Auth tasks, 1 eKYC task, 1 Registration task
               RDS Primary, ElastiCache Primary

T=0–30s ALB detects AZ-a tasks unhealthy
        Routes all traffic to AZ-b and AZ-c tasks
        4 Auth tasks (2 in AZ-b, 2 in AZ-c) absorb load

T=30s   ElastiCache: Replica in AZ-b promoted to Primary
        Redis available (30-60s downtime for Redis failover)

T=60s   RDS: Standby in AZ-b promoted to Primary
        DNS updated, applications reconnect (~90s total)

T=90s   ECS auto-scaling detects reduced capacity
        Launches additional tasks in AZ-b and AZ-c

T=120s  Full capacity restored across surviving AZs

CITIZEN IMPACT:
  Verification API: ~30s degraded (slower, fewer tasks)
  OTP delivery: 30-60s interruption (Redis failover)
  Enrollment: ~90s interruption (RDS failover)
  No data loss (DynamoDB and S3 are AZ-independent)
```

### Scenario 3 — ABIS Vendor Outage

```
TIMELINE:
T=0s    ABIS external system becomes unavailable
T=30s   ABIS queue (nbis-abis-requests) messages time out
T=30s   Deduplication Service cannot get ABIS results
T=30s   Messages retry → fail → stay in queue

ENROLLMENT IMPACT:
  New enrollments: accepted and queued ✅
  Biometric quality checks: continue ✅
  Deduplication: PAUSED (messages wait in SQS)
  UIN assignment: PAUSED (no dedup result)

NO CITIZEN DATA LOST:
  SQS retains messages for 7 days
  When ABIS recovers → processing resumes automatically
  Enrollment packets not lost, just delayed

VERIFICATION IMPACT: ZERO
  Verification uses EXISTING templates in S3
  ABIS is only needed for NEW enrollment deduplication
  All existing citizens can still authenticate

T=4h    ABIS vendor resolves outage
T=4h    Messages in nbis-abis-requests queue process
T=6h    Backlog cleared, normal operation resumes

OPERATIONS RESPONSE:
  DLQ alarm monitors for ABIS timeout errors
  Operations team notified → contacts ABIS vendor
  No manual data recovery needed
```

---

## 15.12 Load Testing for HA Validation

Before going live, HA must be proven through testing:

```
┌──────────────────────────────────────────────────────────────┐
│              HA Validation Tests for NBIS                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Test 1: Normal load test                                    │
│  Tool: Apache JMeter / k6                                    │
│  Load: 1,000 auth requests/sec for 1 hour                   │
│  Pass criteria:                                              │
│  ├── 99.99% requests succeed                               │
│  ├── P95 latency < 200ms                                    │
│  └── P99 latency < 500ms                                    │
│                                                              │
│  Test 2: Spike test                                          │
│  Normal: 100 req/s → Spike to 5,000 req/s in 10 seconds    │
│  Pass criteria:                                              │
│  ├── Auto-scaling kicks in within 60 seconds               │
│  ├── No requests dropped (queue absorbs burst)              │
│  └── System returns to normal after spike                   │
│                                                              │
│  Test 3: Chaos engineering (AZ failure simulation)          │
│  Tool: AWS Fault Injection Simulator (FIS)                  │
│  Action: Terminate all ECS tasks in one AZ                  │
│  Pass criteria:                                             │
│  ├── ALB reroutes within 30 seconds                        │
│  ├── ECS replaces tasks within 120 seconds                 │
│  └── Zero 5XX errors after 30 seconds                      │
│                                                              │
│  Test 4: Database failover test                              │
│  Action: Force RDS failover (reboot with failover)          │
│  Pass criteria:                                              │
│  ├── RDS failover completes in < 60 seconds                │
│  ├── Application reconnects automatically                  │
│  └── No data loss or corruption                            │
│                                                              │
│  Test 5: Cache failure test                                  │
│  Action: Terminate ElastiCache primary node                 │
│  Pass criteria:                                             │
│  ├── Redis failover in < 60 seconds                        │
│  ├── OTP generation continues after failover               │
│  └── Session data intact on replica                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 15.13 SLA Monitoring and Alerting

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS HA Monitoring Stack                        │
└──────────────────────────────────────────────────────────────┘

SYNTHETIC MONITORING (every 60 seconds):
  CloudWatch Synthetics Canary:
  Calls POST /v1/auth/otp/send with test UIN
  ├── Expects HTTP 200 within 500ms
  ├── Fails → CloudWatch alarm → SNS → PagerDuty
  └── Runs from multiple AZs independently

REAL-TIME METRICS (CloudWatch Dashboard):
  ┌──────────────────────────────────────────────────┐
  │  Auth API availability:      99.99%  [GREEN]     │
  │  Auth API P95 latency:       143ms   [GREEN]     │
  │  Auth API P99 latency:       287ms   [GREEN]     │
  │  Active ECS tasks:           6/6     [GREEN]     │
  │  DynamoDB error rate:        0.00%   [GREEN]     │
  │  Redis hit rate:             98.7%   [GREEN]     │
  │  SQS DLQ depth:              0       [GREEN]     │
  │  RDS CPU:                    23%     [GREEN]     │
  │  RDS connections:            45/500  [GREEN]     │
  └──────────────────────────────────────────────────┘

ALARM HIERARCHY:
  P1 (immediate page, 24/7):
  ├── Auth API availability < 99%
  ├── Any DLQ depth > 0
  ├── RDS primary unreachable
  ├── ECS desired count != running count for > 5min
  └── Synthetic canary failing for > 2 consecutive runs

  P2 (notify team, business hours):
  ├── Auth API P95 latency > 500ms
  ├── Redis hit rate < 80%
  ├── RDS CPU > 80% for 10 minutes
  └── ECS task count < desired for < 5 minutes

  P3 (ticket, next business day):
  ├── Auto-scaling triggered (informational)
  ├── Cache evictions detected
  └── Scheduled scaling events completed
```

---

## 15.14 Deployment Without Downtime

High availability extends to the deployment process itself. NBIS uses **blue/green deployment** to release new versions with zero downtime:

```
┌──────────────────────────────────────────────────────────────┐
│              Blue/Green Deployment in NBIS                   │
└──────────────────────────────────────────────────────────────┘

SETUP:
  Blue = current production (Auth Service v2.1)
  Green = new version being deployed (Auth Service v2.2)

STEP 1: Deploy green alongside blue
  ECS launches v2.2 tasks (green)
  ALB: 100% traffic still to blue (v2.1)
  Green tasks warm up, pass health checks

STEP 2: Smoke test green (optional manual verification)
  Send 5% of traffic to green
  Monitor error rate, latency
  All good → proceed

STEP 3: Shift traffic
  ALB: shift to 100% green (v2.2)
  Blue tasks still running (immediate rollback possible)

STEP 4: Monitor
  5 minutes: watch error rates, latency
  If issues detected → ALB: shift 100% back to blue

STEP 5: Drain and decommission blue
  Blue tasks receive no new traffic
  In-flight requests complete
  Blue tasks stopped and decommissioned

RESULT:
  ✅ Zero downtime deployment
  ✅ Instant rollback capability during step 4
  ✅ No citizen impact during entire process

ECS CONFIGURATION:
  "deploymentController": { "type": "CODE_DEPLOY" }
  CodeDeploy manages the blue/green shift
  with configurable traffic shifting strategy
```

---

## 15.15 Connection Pool Management

A commonly overlooked HA concern — connection pools to databases must be sized correctly:

```
┌──────────────────────────────────────────────────────────────┐
│              Connection Pool Configuration                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM:                                                    │
│  Auth Service: 6 tasks × 20 connections = 120 connections   │
│  Auto-scales to 50 tasks × 20 connections = 1,000 conns     │
│  RDS max connections: 500                                   │
│  → Connection exhaustion → 500 errors → outage             │
│                                                              │
│  SOLUTION — RDS Proxy:                                       │
│  Auth Service → RDS Proxy → RDS Primary                     │
│  RDS Proxy pools and multiplexes connections:               │
│  1,000 application connections → 50 actual DB connections   │
│                                                              │
│  Benefits:                                                   │
│  ├── Applications can scale freely                          │
│  ├── RDS not overwhelmed by connection count               │
│  ├── Faster failover (proxy manages reconnection)          │
│  └── IAM authentication for DB connections                 │
│                                                              │
│  Configuration:                                              │
│  RDS Proxy max connections: 90% of RDS max                  │
│  Connection borrow timeout: 30 seconds                      │
│  Idle connection timeout: 1800 seconds                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 15.16 Complete NBIS HA Architecture

```
┌──────────────────────────────────────────────────────────────┐
│          Complete NBIS High Availability Architecture        │
└──────────────────────────────────────────────────────────────┘

                        INTERNET
                           │
                    CloudFront (global edge, DDoS protection)
                           │
                    API Gateway (multi-AZ, managed, 99.95% SLA)
                           │
                    WAF + Shield Advanced
                           │
                    ┌──────┴──────┐
                    │  VPC Link   │
                    └──────┬──────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
        AZ-a              AZ-b              AZ-c
     ┌────────┐        ┌────────┐        ┌────────┐
     │  ECS   │        │  ECS   │        │  ECS   │
     │ Tasks  │        │ Tasks  │        │ Tasks  │
     │ Auth×2 │        │ Auth×2 │        │ Auth×2 │
     │ eKYC×1 │        │ eKYC×1 │        │ eKYC×1 │
     └────────┘        └────────┘        └────────┘
         │                 │                 │
         └─────────────────┼─────────────────┘
                           │ Internal ALB (multi-AZ)
                           │
              ┌────────────┼─────────────┐
              │            │             │
         ┌────┴────┐  ┌────┴────┐  ┌────┴────┐
         │DynamoDB │  │  RDS    │  │   S3    │
         │(multi-AZ│  │Primary  │  │(11 9s   │
         │ built-in│  │Standby→ │  │durability│
         │    )    │  │AZ-b     │  │)        │
         └─────────┘  └─────────┘  └─────────┘
                           │
              ┌────────────┼────────────┐
              │                         │
       ┌──────┴──────┐           ┌──────┴──────┐
       │ ElastiCache │           │  SQS Queues │
       │  Primary AZa│           │  (multi-AZ  │
       │  Replica AZb│           │   built-in) │
       └─────────────┘           └─────────────┘
              │
       CloudWatch + X-Ray + GuardDuty
       (observability and threat detection)
```

---

## 15.17 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP High Availability Reference               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP Kubernetes deployment HA:                            │
│  ├── Multiple pods per service (HPA — Horizontal Pod        │
│  │   Autoscaler equivalent to ECS auto-scaling)            │
│  ├── Pod anti-affinity rules (spread across nodes/AZs)     │
│  │   equivalent to our ECS spread placement constraint     │
│  ├── Kubernetes liveness + readiness probes                │
│  │   equivalent to our ALB health checks                  │
│  └── PodDisruptionBudget: ensures minimum pods during      │
│      rolling updates (equivalent to minimumHealthyPercent)  │
│                                                              │
│  MOSIP HA targets:                                          │
│  ├── Registration Processor: > 99.9%                       │
│  ├── ID Authentication: > 99.99%                           │
│  └── eSignet: > 99.99%                                     │
│                                                              │
│  In AWS deployments of MOSIP:                               │
│  Kubernetes HPA → ECS Target Tracking Auto-scaling         │
│  Pod anti-affinity → ECS AZ spread placement               │
│  Readiness probe → ALB target group health check           │
│  PodDisruptionBudget → minimumHealthyPercent: 100          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 15.18 Key Terms

| Term | Definition |
|------|-----------|
| **High Availability** | System's ability to remain operational despite component failures |
| **SLA** | Service Level Agreement — contractual uptime commitment |
| **Availability Zone** | Physically separate data center within an AWS region |
| **Multi-AZ** | Deploying across multiple AZs so one AZ failure does not cause outage |
| **Auto-scaling** | Automatically adjusting capacity based on current demand |
| **Target tracking** | Auto-scaling that maintains a metric at a target value |
| **Step scaling** | Auto-scaling that scales in defined steps based on metric thresholds |
| **Scheduled scaling** | Pre-scaling at known times before predicted traffic spikes |
| **Blue/green deployment** | Running old and new version simultaneously, shifting traffic safely |
| **RDS Multi-AZ** | RDS configuration with synchronous standby in a different AZ |
| **RDS Proxy** | Connection pooler between application and RDS — handles scaling and failover |
| **Read replica** | Additional RDS instance for read-only queries (async replication) |
| **Strongly consistent read** | DynamoDB read that always returns the most recent data |
| **Eventually consistent read** | DynamoDB read that may return slightly stale data (cheaper) |
| **Chaos engineering** | Deliberately injecting failures to test HA and recovery |
| **AWS FIS** | Fault Injection Simulator — AWS tool for chaos engineering |
| **Synthetic monitoring** | Automated scripts that simulate user actions to test availability |
| **Canary** | CloudWatch Synthetics test — probes endpoints from multiple locations |
| **Circuit breaker** | Stops calls to a failing service to prevent cascade failures |
| **Connection pool** | Reusable database connections shared across application instances |
| **P95/P99 latency** | 95th/99th percentile response time — captures tail latency |

---

## 15.19 Key Takeaways

- **High availability is a government obligation in NBIS** — when the verification API is down, citizens cannot access hospitals, banks, or borders. 99.99% is the minimum for the verification API.
- **Multi-AZ is the foundation** — every component (ECS, RDS, ElastiCache, ALB) must span at least two AZs. A single AZ failure should never cause a full outage.
- **Auto-scaling handles load spikes automatically** — national enrollment campaigns and end-of-month banking spikes are predictable. Use scheduled scaling for known events and target tracking for unknown spikes.
- **Blue/green deployment eliminates deployment downtime** — rolling updates with minimumHealthyPercent: 100 mean old tasks stay up until new tasks are fully healthy.
- **RDS Proxy is mandatory at scale** — without it, auto-scaling ECS tasks will exhaust RDS connections. RDS Proxy multiplexes thousands of app connections into dozens of DB connections.
- **DynamoDB strongly consistent reads for auth** — the identity status must be current. A stale "ACTIVE" status for a suspended identity is a security failure. Pay the extra capacity unit.
- **Chaos engineering validates HA** — test your AZ failure scenario in staging before going live. AWS FIS makes this safe and controlled. Do not discover HA gaps in production.
- **Monitoring must be proactive** — synthetic canaries test the system every minute. Real outages are detected before the first citizen complaint.

---

## 15.20 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 16 | Disaster Recovery — RTO, RPO, backup, cross-region, DR testing |
| Chapter 73 | Monitoring — CloudWatch dashboards, alarms, X-Ray tracing |
| Chapter 74 | Alerting — PagerDuty, SNS, escalation policies, on-call runbooks |

---

*Chapter 15 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
