# Chapter 11 — Event-Driven Architecture

> **Part II — System Architecture**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 11.1 What is Event-Driven Architecture?

**Event-Driven Architecture (EDA)** is a design pattern where services communicate by **producing and consuming events** rather than calling each other directly.

An **event** is a record of something that happened:
- `ENROLLMENT_PACKET_RECEIVED`
- `BIOMETRIC_QUALITY_PASSED`
- `DEDUPLICATION_CLEARED`
- `UIN_ASSIGNED`
- `CREDENTIAL_ISSUED`
- `CITIZEN_DECEASED`

The service that creates the event is the **producer**. The service that reacts to it is the **consumer**. They never call each other directly — the event travels through a **message broker** (SQS, EventBridge, Kafka) between them.

```
┌──────────────────────────────────────────────────────────────┐
│         Request-Driven vs Event-Driven                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  REQUEST-DRIVEN (synchronous):                               │
│                                                              │
│  Registration ──── HTTP POST ────► Biometric Service        │
│  Service           (waits...)      (must be UP)             │
│                    ◄── response ──                           │
│                                                              │
│  Problems:                                                   │
│  ├── Registration must wait for Biometric to respond        │
│  ├── If Biometric is down → Registration fails              │
│  ├── Biometric must handle all Registration traffic         │
│  └── Tight coupling — Registration knows about Biometric    │
│                                                              │
│  ─────────────────────────────────────────────────────────  │
│                                                              │
│  EVENT-DRIVEN (asynchronous):                                │
│                                                              │
│  Registration ──► SQS Queue ──► Biometric Service           │
│  Service          (returns        (picks up when ready)     │
│                   immediately)                              │
│                                                              │
│  Benefits:                                                   │
│  ├── Registration returns immediately (fast response)       │
│  ├── If Biometric is down → events queue up, no data loss   │
│  ├── Biometric scales independently                         │
│  └── Loose coupling — Registration knows only the queue     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 11.2 Why NBIS Must Be Event-Driven

The enrollment write path in NBIS is the textbook case for event-driven design:

```
┌──────────────────────────────────────────────────────────────┐
│         Why Enrollment Must Be Async                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Enrollment involves:                                        │
│  ├── Packet validation (seconds)                            │
│  ├── Biometric quality check (seconds)                      │
│  ├── 1:N ABIS search against millions of records (minutes)  │
│  ├── Manual review if duplicate found (hours/days)          │
│  └── UIN assignment + credential generation (seconds)       │
│                                                              │
│  Total time: minutes to hours                               │
│                                                              │
│  If this were synchronous:                                   │
│  ├── Enrollment officer waits hours at their workstation    │
│  ├── HTTP connection times out                              │
│  ├── Network interruption = lost work                       │
│  └── System cannot handle multiple simultaneous enrollments │
│                                                              │
│  Event-driven solution:                                      │
│  ├── Enrollment officer submits packet                      │
│  ├── Registration Service responds in < 2 seconds:          │
│  │   "Received. Tracking ID: PKT-20250115-001"              │
│  ├── Processing happens in background via events            │
│  └── Citizen notified by SMS when UIN is ready              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 11.3 Core Concepts

### 11.3.1 Event

An **event** is an immutable record of something that happened in the past. It has:

```
┌──────────────────────────────────────────────────────────────┐
│                    Anatomy of an Event                       │
└──────────────────────────────────────────────────────────────┘
{
  "eventId":       "EVT-20250115-abc123",    ← unique ID
  "eventType":     "ENROLLMENT_RECEIVED",    ← what happened
  "eventVersion":  "1.0",                   ← schema version
  "timestamp":     "2025-01-15T10:30:00Z",  ← when it happened
  "source":        "registration-service",  ← who produced it
  "correlationId": "PKT-20250115-001",      ← links related events
  "payload": {                              ← what happened (data)
    "packetId":    "PKT-20250115-001",
    "centerId":    "CENTER-001",
    "operatorId":  "OFR-007",
    "receivedAt":  "2025-01-15T10:30:00Z"
  }
}
```

**Key property:** Events describe what **happened** — past tense. They are facts. They are never modified after creation.

---

### 11.3.2 Producer

A **producer** (also called publisher) is a service that creates and publishes events to the message broker when something significant happens.

```
Registration Service (producer):

packet = receive_enrollment_packet()
save_to_s3(packet)
update_status(packet.id, "RECEIVED")

// Publish event to SQS
sqs.send_message(
  queue_url = "enrollment-packets-queue",
  message_body = json({
    eventType: "ENROLLMENT_RECEIVED",
    packetId: packet.id,
    centerId: packet.centerId
  })
)

return { status: "RECEIVED", trackingId: packet.id }
// Returns immediately — does not wait for biometric check
```

---

### 11.3.3 Consumer

A **consumer** (also called subscriber) is a service that listens to the message broker and reacts when an event arrives.

```
Biometric Service (consumer):

// Continuously polls SQS for new messages
while True:
  messages = sqs.receive_messages(
    queue_url = "enrollment-packets-queue",
    max_messages = 10,
    wait_time_seconds = 20  // long polling
  )

  for message in messages:
    event = parse(message.body)

    if event.eventType == "ENROLLMENT_RECEIVED":
      packet = fetch_from_s3(event.packetId)
      quality_result = run_quality_check(packet.biometrics)

      if quality_result.passed:
        publish_event("BIOMETRIC_QUALITY_PASSED", event.packetId)
      else:
        publish_event("BIOMETRIC_QUALITY_FAILED", event.packetId)

      sqs.delete_message(message.receipt_handle)
      // Only delete AFTER successful processing
```

---

### 11.3.4 Message Broker

The **message broker** sits between producers and consumers. It stores events until a consumer is ready to process them.

```
┌──────────────────────────────────────────────────────────────┐
│              Message Brokers in NBIS                         │
├────────────────────┬─────────────────────────────────────────┤
│ AWS SQS            │ Primary broker for enrollment pipeline  │
│                    │ Queue per processing stage              │
│                    │ Fully managed, scales automatically     │
├────────────────────┼─────────────────────────────────────────┤
│ AWS EventBridge    │ External event integration              │
│                    │ Civil registry events                   │
│                    │ Fan-out (one event → many consumers)    │
├────────────────────┼─────────────────────────────────────────┤
│ AWS SNS            │ Notification fan-out                   │
│                    │ One publish → SMS + email + SQS         │
└────────────────────┴─────────────────────────────────────────┘
```

---

## 11.4 NBIS Event Pipeline — Full Flow

The complete enrollment pipeline is a chain of events, each triggering the next stage:

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS Enrollment Event Chain                     │
└──────────────────────────────────────────────────────────────┘

STAGE 1
Enrollment Station uploads packet
         │
         ▼ publishes
┌─────────────────────────┐
│ ENROLLMENT_RECEIVED     │ → Registration Service → S3
│ { packetId, centerId }  │   saves packet, returns tracking ID
└──────────┬──────────────┘
           │ consumed by
           ▼
STAGE 2: Biometric Service
  Decrypts packet, runs quality check
         │
         ├──► quality FAIL
         │    ▼ publishes
         │    BIOMETRIC_QUALITY_FAILED → notify enrollment officer
         │
         └──► quality PASS
              ▼ publishes
┌─────────────────────────┐
│ BIOMETRIC_QUALITY_PASSED│
│ { packetId, templates } │
└──────────┬──────────────┘
           │ consumed by
           ▼
STAGE 3: Deduplication Service
  Sends to ABIS, waits for result
         │
         ├──► ABIS returns MATCH
         │    ▼ publishes
         │    DEDUPLICATION_FLAGGED → supervisor review queue
         │
         └──► ABIS returns NO MATCH
              ▼ publishes
┌─────────────────────────┐
│ DEDUPLICATION_CLEARED   │
│ { packetId }            │
└──────────┬──────────────┘
           │ consumed by
           ▼
STAGE 4: UIN Service
  Generates UIN, writes to CIDR
         │
         ▼ publishes
┌─────────────────────────┐
│ UIN_ASSIGNED            │
│ { packetId, uin }       │
└──────────┬──────────────┘
           │ consumed by (multiple consumers — fan-out)
           ├──────────────────────────┐
           ▼                          ▼
STAGE 5a: Credential Service   STAGE 5b: Notification Service
  Generate smart card,           Send SMS to citizen:
  QR code, VC                    "Your National ID (UIN:
         │                        123456789012) is ready"
         ▼ publishes
┌─────────────────────────┐
│ CREDENTIAL_ISSUED       │
│ { uin, credentialRef }  │
└─────────────────────────┘
           │ consumed by
           ▼
STAGE 6: Notification Service
  "Your ID card is ready for collection"
```

---

## 11.5 SQS Queue Design for NBIS

Each stage in the pipeline has its own dedicated SQS queue:

```
┌──────────────────────────────────────────────────────────────┐
│                 NBIS SQS Queue Inventory                     │
├────────────────────────────┬─────────────────────────────────┤
│ Queue Name                 │ Purpose                        │
├────────────────────────────┼─────────────────────────────────┤
│ nbis-enrollment-received   │ New packets from Registration  │
│ nbis-biometric-processing  │ Quality check + extraction     │
│ nbis-abis-requests         │ Jobs sent to ABIS (1:N dedup)  │
│ nbis-abis-results          │ Results returned from ABIS     │
│ nbis-dedup-review          │ Flagged duplicates for review  │
│ nbis-uin-assignment        │ Cleared packets for UIN assign │
│ nbis-credential-issuance   │ UIN assigned → make credential │
│ nbis-notifications         │ All citizen notifications      │
│ nbis-civil-registry-events │ Birth/death from civil reg.    │
├────────────────────────────┼─────────────────────────────────┤
│ DLQs (one per queue above) │                                │
│ nbis-enrollment-received-dlq│ Failed enrollment packets     │
│ nbis-biometric-dlq         │ Failed quality checks          │
│ ... (one DLQ per queue)    │                                │
└────────────────────────────┴─────────────────────────────────┘
```

---

## 11.6 Dead Letter Queues (DLQ)

A **Dead Letter Queue** is a safety net — if a message fails processing after a defined number of retries, it is automatically moved to the DLQ instead of being lost or stuck in an infinite retry loop.

```
┌──────────────────────────────────────────────────────────────┐
│                 DLQ Flow in NBIS                             │
└──────────────────────────────────────────────────────────────┘

Normal flow:
  SQS message arrives
        │
  Consumer picks up
        │
  Processing succeeds
        │
  Consumer deletes message ← message gone (processed)

Failure flow:
  SQS message arrives
        │
  Consumer picks up
        │
  Processing FAILS (exception, timeout, ABIS down)
        │
  Message returns to queue (visibility timeout expires)
        │
  Consumer picks up again → FAILS again
        │
  After maxReceiveCount (e.g. 3 attempts):
        │
        ▼
  Message moved to DLQ automatically
        │
  CloudWatch Alarm fires:
  "DLQ nbis-biometric-dlq has 1 message"
        │
  SNS notification → Operations team (email / PagerDuty)
        │
  Engineer investigates:
  ├── Why did processing fail?
  ├── Fix the bug / restore ABIS connectivity
  └── Replay message from DLQ back to main queue
```

### DLQ configuration

```
┌──────────────────────────────────────────────────────────────┐
│                 DLQ Configuration Rules                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  maxReceiveCount:    3                                       │
│  (retry 3 times before DLQ)                                 │
│                                                              │
│  DLQ retention:      14 days                                 │
│  (enough time for investigation and replay)                 │
│                                                              │
│  DLQ alarm:          threshold = 1 message                  │
│  (alert immediately on first failure)                       │
│                                                              │
│  DLQ access:         operations team only (IAM policy)      │
│  (cannot be accidentally deleted by services)               │
│                                                              │
│  CRITICAL RULE:                                              │
│  An enrollment packet in a DLQ = a citizen without          │
│  an identity. Every DLQ message is a P1 incident.           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 11.7 Idempotency — The Critical Property

**Idempotency** means that processing the same message multiple times produces the same result as processing it once.

This is not optional in an event-driven system. Networks are unreliable. Messages can be delivered more than once. A consumer can crash after processing but before deleting the message. **You will receive duplicate messages. Your system must handle them safely.**

```
┌──────────────────────────────────────────────────────────────┐
│              Why Duplicates Happen                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Scenario:                                                   │
│                                                              │
│  1. UIN Service receives DEDUPLICATION_CLEARED event        │
│  2. Assigns UIN 123456789012 to packet PKT-001              │
│  3. Writes record to DynamoDB ✅                            │
│  4. CRASHES before deleting message from SQS               │
│                                                              │
│  5. SQS visibility timeout expires                          │
│  6. Message becomes visible again                           │
│  7. UIN Service picks up the SAME message again             │
│  8. WITHOUT idempotency:                                    │
│     → Assigns ANOTHER UIN 987654321098 to PKT-001          │
│     → Citizen now has TWO UINs ← catastrophic              │
│                                                              │
│  WITH idempotency:                                           │
│  8. UIN Service checks:                                     │
│     "Has PKT-001 been processed before?"                    │
│     DynamoDB GetItem(processedPackets, PKT-001) → FOUND     │
│  9. Returns existing UIN 123456789012                       │
│  10. Deletes message from SQS                               │
│      ← safe, correct, no duplicate                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Idempotency implementation

```
┌──────────────────────────────────────────────────────────────┐
│              Idempotency Pattern                             │
└──────────────────────────────────────────────────────────────┘

// DynamoDB table: processed-events
// Partition key: eventId (or packetId + eventType)
// TTL: 7 days (auto-delete old records)

function process_event(event):

  // Step 1: Check if already processed
  existing = dynamodb.get_item(
    table = "processed-events",
    key   = { "eventId": event.eventId }
  )

  if existing:
    log("Duplicate event detected: " + event.eventId)
    return existing.result  // Return previous result
    // Do NOT process again

  // Step 2: Process the event (business logic)
  result = do_actual_work(event)

  // Step 3: Record as processed (atomic with work)
  dynamodb.transact_write([
    put_item("cidr-table", identity_record),    // actual work
    put_item("processed-events", {              // idempotency record
      eventId: event.eventId,
      processedAt: now(),
      result: result
    })
  ])

  return result
```

---

## 11.8 Event Ordering

SQS Standard queues do **not** guarantee message order. FIFO queues do. Choosing correctly matters.

```
┌──────────────────────────────────────────────────────────────┐
│            SQS Standard vs FIFO for NBIS                     │
├──────────────────────┬───────────────────────────────────────┤
│ SQS Standard         │ SQS FIFO                             │
├──────────────────────┼───────────────────────────────────────┤
│ No order guarantee   │ Strict FIFO per message group        │
│ At-least-once        │ Exactly-once processing              │
│ delivery             │                                      │
│ High throughput      │ 3,000 msg/sec per API action         │
│ (nearly unlimited)   │ (300 without batching)               │
│                      │                                      │
│ Use for:             │ Use for:                             │
│ ├── Notifications    │ ├── UIN assignment (must be          │
│ ├── Quality checks   │ │   sequential per packet)           │
│ ├── ABIS requests    │ ├── Credential issuance              │
│ └── Most stages      │ └── Civil registry updates           │
│     (idempotency     │     (order matters for lifecycle)    │
│     handles dupes)   │                                      │
└──────────────────────┴───────────────────────────────────────┘
```

### Message group ID (FIFO)

```
For FIFO queues, use packetId as the MessageGroupId:

sqs.send_message(
  queue_url        = "nbis-uin-assignment.fifo",
  message_body     = json(event),
  message_group_id = event.packetId   // All messages for
                                       // same packet are
                                       // processed in order
)
```

---

## 11.9 Visibility Timeout

When a consumer picks up a message from SQS, the message becomes **invisible** to other consumers for a defined period — the **visibility timeout**. If processing succeeds, the consumer deletes the message. If processing fails or takes too long, the message becomes visible again and another consumer picks it up.

```
┌──────────────────────────────────────────────────────────────┐
│              Visibility Timeout Strategy                     │
├────────────────────────────┬─────────────────────────────────┤
│ Queue                      │ Visibility Timeout              │
├────────────────────────────┼─────────────────────────────────┤
│ nbis-enrollment-received   │ 60 seconds                     │
│ nbis-biometric-processing  │ 120 seconds                    │
│ nbis-abis-requests         │ 600 seconds (ABIS can be slow) │
│ nbis-uin-assignment        │ 30 seconds (fast operation)    │
│ nbis-notifications         │ 60 seconds                     │
└────────────────────────────┴─────────────────────────────────┘

Rule: visibility timeout must be LONGER than your
      expected processing time. If ABIS takes 5 minutes,
      set 10 minutes to avoid premature re-delivery.
```

---

## 11.10 EventBridge — External Event Integration

**Amazon EventBridge** handles events from **outside** the NBIS — civil registry, population registry, partner systems.

```
┌──────────────────────────────────────────────────────────────┐
│           EventBridge Integration Patterns                   │
└──────────────────────────────────────────────────────────────┘

Pattern 1: Civil Registry → NBIS (death notification)

Civil Registry System
      │
      │ HTTP POST to EventBridge
      ▼
EventBridge Event Bus (nbis-external-events)
Event: {
  "source": "civil-registry",
  "detail-type": "CITIZEN_DECEASED",
  "detail": {
    "registryId": "CR-2025-001234",
    "deathDate": "2025-01-15",
    "uin": "123456789012"
  }
}
      │
      │ Rule: route CITIZEN_DECEASED to...
      ├──────────────────────────────────────────┐
      ▼                                          ▼
SQS queue (nbis-revocation)           Lambda (immediate suspend)
Admin Service picks up                Suspends auth instantly
→ Revokes UIN 123456789012            while full revocation runs


Pattern 2: UIN_ASSIGNED → fan-out to multiple consumers

EventBridge Event Bus (nbis-internal-events)
Event: UIN_ASSIGNED { uin, packetId }
      │
      │ Rule: route to ALL subscribers
      ├──────────────────────┬────────────────────────────┐
      ▼                      ▼                            ▼
SQS (credentials)     SQS (notifications)         SQS (analytics)
Credential Service    Notification Service         Reporting Service
→ Issue smart card    → SMS to citizen             → Update dashboards
```

---

## 11.11 Long Polling

SQS consumers use **long polling** to efficiently wait for messages without wasting compute resources or making excessive empty API calls.

```
┌──────────────────────────────────────────────────────────────┐
│              Short Polling vs Long Polling                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Short polling (default, WaitTimeSeconds = 0):              │
│  Consumer: "Any messages?"                                   │
│  SQS: "No."          ← empty response                       │
│  Consumer: "Any messages?" (1 second later)                  │
│  SQS: "No."                                                  │
│  ...repeated 1,000s of times per hour                       │
│  Cost: high API call volume, wasted compute                 │
│                                                              │
│  Long polling (WaitTimeSeconds = 20):                        │
│  Consumer: "Any messages? I'll wait up to 20 seconds."      │
│  SQS: (holds connection for up to 20 seconds)               │
│  SQS: "Yes! Here are 5 messages." (when they arrive)        │
│  Cost: minimal API calls, efficient                         │
│                                                              │
│  NBIS rule: always use long polling (WaitTimeSeconds = 20)  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 11.12 Event Schema Versioning

Events are a public contract between services. When you change an event's structure, you must do it carefully to avoid breaking consumers.

```
┌──────────────────────────────────────────────────────────────┐
│              Event Schema Versioning                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Always include eventVersion in every event:                │
│  { "eventType": "UIN_ASSIGNED", "eventVersion": "1.0" }    │
│                                                              │
│  Safe changes (backward compatible):                        │
│  ✅ Add optional new field to payload                        │
│  ✅ Add new event type                                       │
│  ✅ Increase payload field size                             │
│                                                              │
│  Breaking changes (requires new version):                   │
│  ❌ Remove a field consumers depend on                       │
│  ❌ Change a field's data type                              │
│  ❌ Rename an event type                                    │
│  ❌ Change the meaning of a field                           │
│                                                              │
│  Versioning strategy:                                        │
│  v1.0 consumers → keep working with v1.0 events             │
│  v2.0 events → produced alongside v1.0 during migration     │
│  After all consumers upgraded → stop producing v1.0         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 11.13 Complete NBIS Event Inventory

```
┌──────────────────────────────────────────────────────────────┐
│                 NBIS Event Inventory                         │
├──────────────────────────┬───────────────────────────────────┤
│ Event Type               │ Producer → Consumer(s)           │
├──────────────────────────┼───────────────────────────────────┤
│ ENROLLMENT_RECEIVED      │ Registration → Biometric         │
│ BIOMETRIC_QUALITY_PASSED │ Biometric → Deduplication        │
│ BIOMETRIC_QUALITY_FAILED │ Biometric → Notification         │
│ DEDUP_JOB_SUBMITTED      │ Deduplication → ABIS             │
│ DEDUP_RESULT_RECEIVED    │ ABIS → Deduplication             │
│ DEDUPLICATION_CLEARED    │ Deduplication → UIN Service      │
│ DEDUPLICATION_FLAGGED    │ Deduplication → Review Queue     │
│ MANUAL_REVIEW_CLEARED    │ Supervisor → UIN Service         │
│ MANUAL_REVIEW_REJECTED   │ Supervisor → Notification        │
│ UIN_ASSIGNED             │ UIN → Credential, Notification   │
│ CREDENTIAL_ISSUED        │ Credential → Notification        │
│ CITIZEN_UPDATED          │ Resident Svc → Audit             │
│ BIOMETRIC_LOCKED         │ Resident Svc → Auth, Audit       │
│ BIOMETRIC_UNLOCKED       │ Resident Svc → Auth, Audit       │
│ IDENTITY_SUSPENDED       │ Admin → Auth, Notification       │
│ IDENTITY_REVOKED         │ Admin → Auth, Credential, Notif  │
│ CITIZEN_DECEASED         │ Civil Registry → Admin, Auth     │
│ AUTH_SUCCESS             │ Auth → Audit                     │
│ AUTH_FAILURE             │ Auth → Audit, Fraud Detection    │
│ PARTNER_ONBOARDED        │ Partner Mgmt → API Gateway       │
│ PARTNER_SUSPENDED        │ Partner Mgmt → API Gateway       │
└──────────────────────────┴───────────────────────────────────┘
```

---

## 11.14 Error Handling in Event-Driven Systems

Errors in async pipelines are harder to handle than in sync systems — the caller is long gone when the error occurs.

```
┌──────────────────────────────────────────────────────────────┐
│              Error Handling Strategy                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Transient errors (retry):                                   │
│  ├── Network timeout calling ABIS                           │
│  ├── DynamoDB throttling                                    │
│  ├── S3 temporary unavailability                            │
│  → Retry with exponential backoff                           │
│  → Message stays in queue (visibility timeout)              │
│  → After maxReceiveCount → DLQ                              │
│                                                              │
│  Permanent errors (DLQ + alert):                             │
│  ├── Malformed packet (cannot ever be parsed)               │
│  ├── Missing required field in event payload                │
│  ├── UIN already assigned for this packet                   │
│  → Do not retry endlessly                                   │
│  → Move to DLQ immediately                                  │
│  → Alert operations team                                    │
│  → Manual investigation required                            │
│                                                              │
│  Business errors (publish error event):                      │
│  ├── Biometric quality below threshold                      │
│  ├── Duplicate found in deduplication                       │
│  → Not a system error — expected business outcome           │
│  → Publish BIOMETRIC_QUALITY_FAILED event                   │
│  → Normal pipeline handles it                               │
│  → Do NOT put in DLQ                                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 11.15 Monitoring an Event-Driven System

Observability is harder in async systems because there is no single request to trace. Key metrics and alarms:

```
┌──────────────────────────────────────────────────────────────┐
│              EDA Monitoring in NBIS                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SQS metrics to watch (CloudWatch):                         │
│  ├── ApproximateNumberOfMessagesVisible                     │
│  │   (queue depth — rising = consumer too slow)            │
│  ├── ApproximateAgeOfOldestMessage                         │
│  │   (how long oldest message has been waiting)            │
│  ├── NumberOfMessagesSent (throughput)                      │
│  └── NumberOfMessagesDeleted (processed successfully)       │
│                                                              │
│  Alarms:                                                     │
│  ├── DLQ depth > 0 → P1 alert (operations + on-call)       │
│  ├── Queue age > 10 min → P2 alert (slow consumer)         │
│  ├── Queue depth > 1000 → P2 alert (backlog building)      │
│  └── ABIS queue age > 30 min → P1 alert (ABIS down?)       │
│                                                              │
│  End-to-end tracing:                                         │
│  ├── correlationId links all events for one enrollment      │
│  ├── CloudWatch Logs Insights: search by correlationId      │
│  └── AWS X-Ray: trace async paths across services           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 11.16 Complete AWS Architecture for EDA

```
┌──────────────────────────────────────────────────────────────┐
│          NBIS Event-Driven Architecture on AWS               │
└──────────────────────────────────────────────────────────────┘

EXTERNAL EVENTS:
Civil Registry ──HTTP──► EventBridge Bus (nbis-external)
                              │
                    ┌─────────▼──────────┐
                    │  EventBridge Rules  │
                    └──┬──────────┬───────┘
                       │          │
                       ▼          ▼
              SQS (revoke)  Lambda (suspend)

INTERNAL ENROLLMENT PIPELINE:

Registration Service
      │
      ▼ SendMessage
SQS (nbis-enrollment-received)  ←── DLQ (nbis-enrollment-dlq)
      │
      ▼ trigger (Lambda or ECS polling)
Biometric Service (ECS Fargate)
      │
      ├── FAIL ──► SQS (nbis-biometric-dlq) ──► CloudWatch Alarm
      │
      └── PASS ──► SendMessage
                       │
                       ▼
            SQS (nbis-abis-requests)
                       │
                       ▼
               ABIS (external)
                       │
                       ▼
            SQS (nbis-abis-results)
                       │
                       ▼
           Deduplication Service (Lambda)
                       │
            ┌──────────┴──────────┐
            │                     │
            ▼                     ▼
SQS (nbis-dedup-review)   SQS (nbis-uin-assignment.fifo)
(flagged duplicates)              │
                                  ▼
                          UIN Service (Lambda)
                                  │
                                  ▼ PutEvents
                      EventBridge (nbis-internal)
                                  │
                     ┌────────────┼──────────────┐
                     ▼            ▼              ▼
             SQS(credential) SQS(notif)   SQS(analytics)
             Credential Svc  Notification  Reporting
             → Smart card    → SMS to cit. → Dashboards
```

---

## 11.17 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│          MOSIP Event-Driven Architecture Reference           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP Registration Processor uses Kafka as its             │
│  internal event broker (in self-hosted deployments)         │
│                                                              │
│  MOSIP Kafka topics (equivalent to our SQS queues):         │
│  ├── mosip.registration.processor.packet.receiver           │
│  ├── mosip.registration.processor.packet.validator          │
│  ├── mosip.registration.processor.abis.handler              │
│  ├── mosip.registration.processor.uin.generator             │
│  └── mosip.registration.processor.notification              │
│                                                              │
│  In AWS deployments of MOSIP:                               │
│  Kafka → replaced with Amazon MSK (Managed Kafka)           │
│  or replaced entirely with SQS + EventBridge                │
│                                                              │
│  MOSIP's approach validates our design:                      │
│  ├── Each stage is a separate processor/service             │
│  ├── Stages communicate via topics (events)                 │
│  ├── Each stage is independently scalable                   │
│  └── Failed packets do not block other enrollments          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 11.18 Key Terms

| Term | Definition |
|------|-----------|
| **Event** | Immutable record of something that happened — past tense, never modified |
| **Producer** | Service that creates and publishes events |
| **Consumer** | Service that listens for and reacts to events |
| **Message broker** | Infrastructure that stores and delivers events (SQS, EventBridge, Kafka) |
| **SQS** | Amazon Simple Queue Service — primary NBIS message broker |
| **Dead Letter Queue** | Destination for messages that fail after max retries |
| **Idempotency** | Processing the same message multiple times produces the same result |
| **Visibility timeout** | Period a message is invisible after pickup — allows time for processing |
| **Long polling** | Consumer waits up to 20s for messages — reduces empty API calls |
| **maxReceiveCount** | How many times a message can be received before going to DLQ |
| **EventBridge** | AWS event bus — routes events between services and external systems |
| **FIFO queue** | SQS queue with strict ordering and exactly-once processing |
| **Standard queue** | SQS queue with high throughput, at-least-once delivery, no order |
| **Message group ID** | FIFO key that groups related messages for ordered processing |
| **correlationId** | ID linking all events belonging to one enrollment for tracing |
| **Fan-out** | One event published → multiple consumers receive it simultaneously |
| **Backpressure** | Queue depth rising — consumer cannot keep up with producer rate |
| **Exponential backoff** | Retry delay doubles on each failure — avoids hammering a sick service |

---

## 11.19 Key Takeaways

- **Events are facts** — immutable records of things that happened. They describe the past, not instructions for the future.
- **Producers and consumers never call each other directly** — the queue is the only coupling between them. Remove the queue and neither service knows the other exists.
- **Async is mandatory for enrollment** — the ABIS 1:N search takes minutes. No citizen, no officer, and no HTTP connection should wait that long. Fire the event, return immediately, process in the background.
- **Idempotency is not optional** — duplicate messages will arrive. Your system must detect and safely ignore them. Use a processed-events table with the eventId as the key.
- **Every queue must have a DLQ with an alarm** — a failed enrollment packet is a citizen without an identity. That is a P1 incident. Never let it be silently dropped.
- **Fan-out with EventBridge** — when one event (`UIN_ASSIGNED`) needs to trigger multiple services (Credential, Notification, Analytics), use EventBridge rules to fan out rather than making the producer know about all consumers.
- **Monitor queue depth and message age** — a rising queue is the first warning sign of a slow consumer or a downstream service outage. Alert before it becomes a crisis.
- **correlationId is your debugging lifeline** — every event for a single enrollment carries the same correlationId. When an enrollment fails at 3am, the correlationId is how you find every related log entry across all services.

---

## 11.20 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 12 | Message Queues — SQS deep dive, FIFO vs Standard, visibility timeout tuning |
| Chapter 13 | Service Discovery — Cloud Map, internal ALB, how services find each other |
| Chapter 14 | Caching — ElastiCache Redis for OTP, sessions, and template caching |

---

*Chapter 11 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
