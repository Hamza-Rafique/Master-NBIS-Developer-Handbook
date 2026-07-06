# Chapter 6 — Overall NBIS Architecture

> **Part II — System Architecture**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 6.1 What is System Architecture?

**System architecture** is the high-level blueprint of how all components of a system are structured, how they communicate, and how they share responsibility.

Before writing a single line of code, a good engineer draws the architecture. It answers:

```
┌──────────────────────────────────────────────────────────────┐
│              Architecture answers these questions            │
├──────────────────────────────────────────────────────────────┤
│  What components exist?                                      │
│  What is each component responsible for?                     │
│  How do components communicate?                              │
│  Where does data live?                                       │
│  Where are the security boundaries?                          │
│  How does the system scale?                                  │
│  What happens when one component fails?                      │
└──────────────────────────────────────────────────────────────┘
```

This chapter gives you the **complete picture** of an NBIS — every layer, every component, every data flow, mapped to AWS services.

---

## 6.2 The Two Paths: Write vs Read

The single most important mental model for NBIS architecture is the distinction between the **write path** and the **read path**.

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   WRITE PATH (Enrollment)         READ PATH (Verification)  │
│   ────────────────────────        ─────────────────────────  │
│   Happens ONCE per citizen        Happens MILLIONS of times  │
│   Complex, multi-step             Simple, fast               │
│   Asynchronous (takes minutes)    Synchronous (milliseconds) │
│   High data volume (biometrics)   Low data volume (scores)   │
│   Runs on SQS + Lambda            Runs on API Gateway        │
│   Involves ABIS (external)        Involves CIDR only         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

Design the write path for **correctness**. Design the read path for **speed**. They have fundamentally different requirements and should be built as separate service groups.

---

## 6.3 Overall Architecture — Full System Diagram

```
                    ┌──────────────────────────────────────┐
                    │         EXTERNAL WORLD               │
                    │                                      │
                    │  Citizens    Relying Parties         │
                    │  (enroll)    (banks, hospitals)      │
                    └──────┬───────────────┬───────────────┘
                           │               │
                    ┌──────▼───────────────▼───────────────┐
                    │         API GATEWAY LAYER            │
                    │                                      │
                    │  Rate limiting · Auth · Routing      │
                    │  TLS termination · Audit log         │
                    │                                      │
                    │  AWS: API Gateway + WAF + CloudFront │
                    └──────┬───────────────┬───────────────┘
                           │               │
              ┌────────────▼──┐         ┌──▼─────────────────┐
              │  WRITE PATH   │         │   READ PATH        │
              │  (Enrollment) │         │   (Verification)   │
              └──────┬────────┘         └──┬─────────────────┘
                     │                     │
    ┌────────────────▼──────┐   ┌──────────▼──────────────────┐
    │  ENROLLMENT SERVICES  │   │  VERIFICATION SERVICES      │
    │                       │   │                             │
    │  Pre-Registration     │   │  Authentication Service     │
    │  Registration Client  │   │  eKYC Service               │
    │  Registration         │   │  eSignet (OIDC)             │
    │  Processor            │   │  Resident Services          │
    │                       │   │                             │
    │  AWS: ECS Fargate     │   │  AWS: ECS Fargate           │
    └──────────┬────────────┘   └──────────┬──────────────────┘
               │                            │
    ┌──────────▼────────────┐               │
    │    MESSAGE QUEUE      │               │
    │                       │               │
    │  Enrollment packets   │               │
    │  Dedup triggers       │               │
    │  Notification events  │               │
    │                       │               │
    │  AWS: SQS             │               │
    └──────────┬────────────┘               │
               │                            │
    ┌──────────▼────────────┐               │
    │  PROCESSING LAYER     │               │
    │                       │               │
    │  Quality checks       │               │
    │  Deduplication        │               │
    │  UIN assignment       │               │
    │  Credential issuance  │               │
    │                       │               │
    │  AWS: Lambda + ECS    │               │
    └──────────┬────────────┘               │
               │                            │
    ┌──────────▼────────────────────────────▼──────────────────┐
    │                CENTRAL IDENTITY DATA REPOSITORY          │
    │                        (CIDR)                            │
    │                                                          │
    │  Citizen records · Biometric templates · Credentials     │
    │  Audit logs · Enrollment history                         │
    │                                                          │
    │  AWS: DynamoDB + RDS + S3 + KMS                          │
    └──────────────────────┬───────────────────────────────────┘
                           │
    ┌──────────────────────▼───────────────────────────────────┐
    │              EXTERNAL INTEGRATIONS                       │
    │                                                          │
    │  ABIS (biometric engine) · Civil Registry               │
    │  Credential vendor · Population Registry                │
    │                                                          │
    │  AWS: SQS (ABIS queue) · EventBridge (civil reg.)       │
    └──────────────────────────────────────────────────────────┘
```

---

## 6.4 Layer 1 — API Gateway Layer

The **API Gateway** is the single entry point for all traffic into the NBIS. No service is ever exposed directly to the internet.

```
┌──────────────────────────────────────────────────────────────┐
│                    API Gateway Layer                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Responsibilities:                                           │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 1. TLS termination (HTTPS enforcement)                 │  │
│  │ 2. Authentication (Cognito JWT validation)             │  │
│  │ 3. Authorization (check role before routing)           │  │
│  │ 4. Rate limiting (per client, per endpoint)            │  │
│  │ 5. Request routing (enrollment vs verification)        │  │
│  │ 6. DDoS protection (AWS WAF + Shield)                  │  │
│  │ 7. Request/response logging (every call audited)       │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  AWS Services:                                               │
│  ┌─────────────────────────────────┐                         │
│  │ API Gateway (routing + throttle)│                         │
│  │ WAF (web application firewall)  │                         │
│  │ CloudFront (edge caching + DDoS)│                         │
│  │ Cognito (JWT validation)        │                         │
│  └─────────────────────────────────┘                         │
└──────────────────────────────────────────────────────────────┘
```

### Routing rules example

```
POST /enrollment/register      →  Registration Service
POST /enrollment/biometrics    →  Biometric Service
GET  /auth/verify              →  Verification Service
POST /auth/ekyc                →  eKYC Service
GET  /resident/history         →  Resident Services
POST /admin/revoke             →  Admin Service (ADMIN role only)
```

---

## 6.5 Layer 2 — Enrollment Services (Write Path)

```
┌──────────────────────────────────────────────────────────────┐
│                   Enrollment Services                        │
├───────────────────────┬──────────────────────────────────────┤
│                       │                                      │
│  Pre-Registration     │  Online appointment booking         │
│  Service              │  Slot management                    │
│                       │  Document checklist                 │
│                       │  AWS: ECS + DynamoDB                │
│                       │                                      │
├───────────────────────┼──────────────────────────────────────┤
│                       │                                      │
│  Registration Client  │  Desktop app at enrollment center   │
│  (edge)               │  Biometric capture (offline capable)│
│                       │  Packet encryption + upload         │
│                       │  AWS: S3 (packet upload)            │
│                       │                                      │
├───────────────────────┼──────────────────────────────────────┤
│                       │                                      │
│  Registration         │  Packet decryption + validation     │
│  Processor            │  Quality check pipeline             │
│  (back-end pipeline)  │  ABIS dedup trigger                 │
│                       │  UIN assignment                     │
│                       │  Credential generation              │
│                       │  AWS: SQS + Lambda + ECS            │
│                       │                                      │
└───────────────────────┴──────────────────────────────────────┘
```

### Enrollment packet flow

```
Enrollment Center
      │
      │  Encrypted enrollment packet
      │  (demographics + biometrics + operator signature)
      ▼
S3 bucket (secure upload)
      │
      ▼ triggers
SQS queue (enrollment-packets)
      │
      ▼ consumed by
Registration Processor (ECS / Lambda)
      │
      ├──► Decrypt packet (KMS)
      ├──► Validate structure
      ├──► Check document authenticity
      ├──► Run biometric quality check
      ├──► Send to ABIS queue for dedup
      └──► On dedup pass → assign UIN → write to CIDR
```

---

## 6.6 Layer 3 — Verification Services (Read Path)

```
┌──────────────────────────────────────────────────────────────┐
│                   Verification Services                      │
├───────────────────────┬──────────────────────────────────────┤
│                       │                                      │
│  Authentication       │  1:1 biometric match                │
│  Service              │  OTP verification                   │
│                       │  Demographic match                  │
│                       │  Returns: match/no-match + score    │
│                       │  AWS: Lambda + DynamoDB GetItem      │
│                       │                                      │
├───────────────────────┼──────────────────────────────────────┤
│                       │                                      │
│  eKYC Service         │  Returns signed attribute packet    │
│                       │  (name, DOB, address, photo)        │
│                       │  Requires citizen consent           │
│                       │  AWS: Lambda + KMS (sign response)  │
│                       │                                      │
├───────────────────────┼──────────────────────────────────────┤
│                       │                                      │
│  eSignet              │  OIDC / OAuth2 identity provider    │
│  (MOSIP product)      │  "Sign in with National ID"         │
│                       │  Issues JWT tokens                  │
│                       │  AWS: ECS + ElastiCache (sessions)  │
│                       │                                      │
├───────────────────────┼──────────────────────────────────────┤
│                       │                                      │
│  Resident Services    │  Self-service portal                │
│                       │  View / update own record           │
│                       │  View auth history                  │
│                       │  Lock / unlock biometrics           │
│                       │  AWS: ECS + DynamoDB                │
│                       │                                      │
└───────────────────────┴──────────────────────────────────────┘
```

---

## 6.7 Layer 4 — Message Queue Layer

The **message queue** is the backbone of the asynchronous write path. It decouples services, absorbs traffic spikes, and ensures no enrollment packet is lost even if a downstream service is temporarily unavailable.

```
┌──────────────────────────────────────────────────────────────┐
│                    SQS Queue Structure                       │
├────────────────────────┬─────────────────────────────────────┤
│ Queue                  │ Purpose                            │
├────────────────────────┼─────────────────────────────────────┤
│ enrollment-packets     │ New enrollment uploads             │
│ abis-dedup-requests    │ Biometric dedup jobs to ABIS       │
│ abis-dedup-results     │ Results back from ABIS             │
│ credential-issuance    │ Trigger credential generation      │
│ notification-events    │ SMS/email to citizen on completion │
│ civil-registry-events  │ Birth / death events from civil reg│
└────────────────────────┴─────────────────────────────────────┘
```

### Dead letter queues

Every SQS queue has a **Dead Letter Queue (DLQ)** — if a message fails processing after N retries, it moves to the DLQ for manual investigation. In an NBIS, a failed enrollment packet must never be silently dropped.

```
enrollment-packets (main queue)
      │
      │ fails after 3 retries
      ▼
enrollment-packets-dlq (dead letter queue)
      │
      ▼ triggers alarm
CloudWatch Alarm
      │
      ▼ notifies
Operations team (SNS → email / PagerDuty)
```

---

## 6.8 Layer 5 — Processing Layer

The **processing layer** is where the complex, asynchronous work happens — quality checks, deduplication, UIN assignment, credential generation.

```
┌──────────────────────────────────────────────────────────────┐
│                    Processing Pipeline                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Step 1: Packet Intake                                       │
│  ├── Decrypt packet (KMS)                                    │
│  ├── Validate digital signature (operator key)              │
│  └── Validate packet structure                              │
│                                                              │
│  Step 2: Quality Checks                                      │
│  ├── Biometric quality score ≥ threshold?                   │
│  ├── Document authenticity check                            │
│  └── Demographic completeness check                         │
│                                                              │
│  Step 3: Deduplication                                       │
│  ├── Send to ABIS via SQS queue                             │
│  ├── Wait for ABIS result (async)                           │
│  ├── If match found → flag for manual review                │
│  └── If no match → proceed to UIN assignment                │
│                                                              │
│  Step 4: UIN Assignment                                      │
│  ├── Generate unique, non-sequential UIN                    │
│  ├── Write identity record to CIDR (DynamoDB)               │
│  └── Write biometric templates to secure store (S3 + KMS)  │
│                                                              │
│  Step 5: Credential Issuance                                 │
│  ├── Generate signed Verifiable Credential (VC)             │
│  ├── Queue physical card for printing                       │
│  └── Notify citizen (SMS / email)                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 6.9 Layer 6 — Central Identity Data Repository (CIDR)

The **CIDR** is the authoritative data store. Every identity record, biometric template, audit log entry, and credential artifact lives here.

```
┌──────────────────────────────────────────────────────────────┐
│              CIDR — Data Store Architecture                  │
├────────────────────┬─────────────────────────────────────────┤
│ Data               │ Storage                  │ Why          │
├────────────────────┼──────────────────────────┼──────────────┤
│ Identity records   │ DynamoDB                 │ Fast 1:1     │
│ (demographics,     │ (partition key = UIN)    │ lookup by UIN│
│ UIN, status)       │                          │              │
├────────────────────┼──────────────────────────┼──────────────┤
│ Biometric          │ S3 (encrypted)           │ Large binary │
│ templates          │ + KMS (key per citizen)  │ objects      │
├────────────────────┼──────────────────────────┼──────────────┤
│ Audit logs         │ CloudWatch Logs          │ Immutable,   │
│                    │ (Object Lock enabled)    │ append-only  │
├────────────────────┼──────────────────────────┼──────────────┤
│ Relational data    │ RDS PostgreSQL           │ Complex      │
│ (enrollment        │                          │ queries,     │
│ history, reports)  │                          │ joins        │
├────────────────────┼──────────────────────────┼──────────────┤
│ Credential         │ S3 (signed artifacts)    │ QR codes,    │
│ artifacts          │                          │ VC JSON      │
└────────────────────┴──────────────────────────┴──────────────┘
```

### Data access patterns

```
Verification (read by UIN):
  DynamoDB GetItem (UIN) → O(1) → < 10ms

Authentication (biometric match):
  DynamoDB GetItem (UIN) → fetch template ref
  S3 GetObject (template) → pass to matcher
  Matcher returns score → apply threshold

Deduplication (1:N search):
  ABIS handles this — not DynamoDB
  ABIS maintains its own optimized index
```

---

## 6.10 Layer 7 — External Integrations

```
┌──────────────────────────────────────────────────────────────┐
│                  External Integrations                       │
├────────────────────┬─────────────────────────────────────────┤
│ System             │ Integration Pattern                     │
├────────────────────┼─────────────────────────────────────────┤
│ ABIS               │ SQS queue (async job/result pattern)   │
│                    │ NBIS never calls ABIS API directly      │
├────────────────────┼─────────────────────────────────────────┤
│ Civil Registry     │ EventBridge (birth/death events)       │
│                    │ Bi-directional: events in, events out   │
├────────────────────┼─────────────────────────────────────────┤
│ Population Registry│ EventBridge (status sync events)       │
├────────────────────┼─────────────────────────────────────────┤
│ Credential vendor  │ SFTP / secure API (card print jobs)    │
├────────────────────┼─────────────────────────────────────────┤
│ Notification (SMS) │ SNS → SMS gateway                      │
├────────────────────┼─────────────────────────────────────────┤
│ Relying parties    │ API Gateway (REST + OIDC)              │
└────────────────────┴─────────────────────────────────────────┘
```

---

## 6.11 Security Architecture

Security is not a layer — it is a **cross-cutting concern** that applies at every layer simultaneously.

```
┌──────────────────────────────────────────────────────────────┐
│                   Security Architecture                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Network layer:                                              │
│  ├── All services in private VPC subnets                    │
│  ├── No public IPs on any microservice                      │
│  ├── Security groups: deny all, allow only needed ports     │
│  └── VPC Flow Logs (all traffic recorded)                   │
│                                                              │
│  Transport layer:                                            │
│  ├── TLS 1.3 on all external connections                    │
│  ├── mTLS between internal microservices                    │
│  └── Certificate rotation via ACM (automated)              │
│                                                              │
│  Application layer:                                          │
│  ├── JWT validation at API Gateway (Cognito)                │
│  ├── RBAC enforcement per endpoint                          │
│  ├── Input validation (no SQL injection / XSS)              │
│  └── Request signing (enrollment operator key)              │
│                                                              │
│  Data layer:                                                 │
│  ├── AES-256 encryption at rest (all stores)                │
│  ├── KMS key per data classification                        │
│  ├── CloudHSM for signing keys (govt holds master key)      │
│  └── Biometric templates: double-encrypted                  │
│                                                              │
│  Audit layer:                                                │
│  ├── CloudTrail (all AWS API calls)                         │
│  ├── CloudWatch Logs (application audit)                    │
│  ├── Object Lock (immutable logs)                           │
│  └── GuardDuty (anomaly detection)                          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 6.12 Scalability Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                  Scalability Design                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Read path (verification) — scale horizontally:             │
│  ├── API Gateway: no limit (managed service)                │
│  ├── Lambda: auto-scales to thousands of concurrent calls   │
│  ├── DynamoDB: auto-scaling, consistent single-digit ms     │
│  └── ElastiCache: cache frequent lookups (session tokens)   │
│                                                              │
│  Write path (enrollment) — scale via queue:                 │
│  ├── SQS: absorbs enrollment bursts (national campaigns)    │
│  ├── Registration Processor: scales ECS tasks with queue    │
│  ├── ABIS: typically fixed capacity — queue manages load    │
│  └── DynamoDB: write capacity auto-scaling on UIN insert    │
│                                                              │
│  Database scaling:                                           │
│  ├── DynamoDB: partition by UIN (even distribution)         │
│  ├── RDS: read replicas for reporting queries               │
│  └── S3: unlimited scale for biometric templates            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 6.13 High Availability Architecture

```
┌──────────────────────────────────────────────────────────────┐
│               High Availability Design                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Multi-AZ deployment (minimum):                             │
│  ├── API Gateway: managed, multi-AZ by default             │
│  ├── ECS Fargate: tasks spread across 3 AZs                │
│  ├── DynamoDB: multi-AZ by default (3 replicas)            │
│  ├── RDS: Multi-AZ standby (automatic failover)            │
│  └── SQS: fully managed, multi-AZ by default               │
│                                                              │
│  Target SLAs:                                               │
│  ├── Verification API:   99.99% uptime   < 200ms response  │
│  ├── eKYC API:           99.99% uptime   < 500ms response  │
│  ├── Enrollment:         99.9% uptime    async (minutes)   │
│  └── Resident portal:    99.9% uptime    < 1s response     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 6.14 Complete AWS Service Map

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS Complete AWS Service Map                   │
├──────────────────────────────┬───────────────────────────────┤
│ Component                    │ AWS Service                   │
├──────────────────────────────┼───────────────────────────────┤
│ API entry point              │ API Gateway                   │
│ DDoS + WAF                   │ WAF + Shield + CloudFront     │
│ Authentication (JWT)         │ Cognito user pools            │
│ Microservices                │ ECS Fargate (containers)      │
│ Async processing             │ Lambda (event-triggered)      │
│ Message queue                │ SQS (standard + FIFO)         │
│ Event bus (civil registry)   │ EventBridge                   │
│ Identity records             │ DynamoDB                      │
│ Relational / reporting data  │ RDS PostgreSQL (Multi-AZ)     │
│ Biometric templates          │ S3 (encrypted, private)       │
│ Credential artifacts         │ S3 (signed VCs, QR codes)     │
│ Encryption keys              │ KMS + CloudHSM                │
│ Notifications                │ SNS                           │
│ Session cache                │ ElastiCache (Redis)           │
│ Audit logs                   │ CloudWatch Logs (Object Lock) │
│ AWS API audit                │ CloudTrail                    │
│ Anomaly detection            │ GuardDuty                     │
│ Metrics + alarms             │ CloudWatch Metrics + Alarms   │
│ Certificate management       │ ACM (public + private CA)     │
│ Secret storage               │ Secrets Manager               │
│ Container registry           │ ECR                           │
│ Cold archival                │ S3 Glacier                    │
└──────────────────────────────┴───────────────────────────────┘
```

---

## 6.15 MOSIP Architecture Mapping

```
┌──────────────────────────────────────────────────────────────┐
│          MOSIP Module → NBIS Layer Mapping                   │
├──────────────────────────┬───────────────────────────────────┤
│ MOSIP Module             │ NBIS Layer                       │
├──────────────────────────┼───────────────────────────────────┤
│ Pre-Registration         │ Enrollment Services              │
│ Registration Client      │ Enrollment Services (edge)       │
│ Registration Processor   │ Processing Layer                 │
│ ID Repository            │ CIDR (DynamoDB + S3)             │
│ ID Authentication        │ Verification Services            │
│ Resident Services        │ Verification Services            │
│ Partner Management       │ API Gateway Layer (RP onboarding)│
│ eSignet                  │ Verification Services (OIDC)     │
│ Inji                     │ Digital credential client app    │
│ Admin Services           │ Management (cross-cutting)       │
└──────────────────────────┴───────────────────────────────────┘
```

---

## 6.16 Key Terms

| Term | Definition |
|------|-----------|
| **Write path** | The enrollment pipeline — complex, async, happens once per citizen |
| **Read path** | The verification pipeline — simple, sync, happens millions of times |
| **API Gateway** | Single entry point — handles auth, routing, rate limiting, logging |
| **CIDR** | Central Identity Data Repository — the authoritative data store |
| **SQS** | Simple Queue Service — decouples services in the async pipeline |
| **Dead Letter Queue** | Queue for messages that fail repeatedly — prevents silent data loss |
| **ECS Fargate** | AWS serverless container service — runs Spring Boot microservices |
| **Lambda** | AWS serverless function — handles async, event-triggered processing |
| **DynamoDB** | AWS NoSQL database — stores identity records with O(1) UIN lookup |
| **CloudHSM** | Hardware Security Module — government holds master encryption keys |
| **Object Lock** | S3 feature making audit logs immutable (WORM — write once, read many) |
| **GuardDuty** | AWS threat detection — flags anomalous access patterns |
| **Multi-AZ** | Deployment across multiple availability zones for high availability |
| **mTLS** | Mutual TLS — both client and server authenticate each other |
| **EventBridge** | AWS event bus — used for civil registry integration events |

---

## 6.17 Key Takeaways

- **Write path and read path are fundamentally different** — design them separately. Enrollment is async and complex; verification is sync and must be fast.
- **API Gateway is the only public face of the system** — no microservice is ever exposed directly to the internet.
- **SQS is the backbone of the write path** — it absorbs bursts (national enrollment campaigns), prevents data loss, and decouples services so they can fail and recover independently.
- **DynamoDB + S3 is the right combination** for the CIDR — DynamoDB for fast identity record lookup by UIN, S3 for large biometric template blobs.
- **Security is cross-cutting** — it applies at network, transport, application, data, and audit layers simultaneously. Missing any one layer is a vulnerability.
- **CloudHSM + government-held keys = data sovereignty** — even though AWS hosts the infrastructure, the government controls the cryptographic keys.
- **Dead letter queues are non-negotiable** — a failed enrollment packet that is silently dropped means a citizen has no identity. Every queue must have a DLQ with an alarm.
- **MOSIP implements this entire architecture** — studying MOSIP module by module is studying this architecture in production code.

---

## 6.18 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 7 | Microservices — how each service is structured, single responsibility, own data store |
| Chapter 8 | API Gateway — deep dive into routing, auth, rate limiting, WAF |
| Chapter 11 | Event-Driven Architecture — SQS patterns, dead letter queues, idempotency |

---

*Chapter 6 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*