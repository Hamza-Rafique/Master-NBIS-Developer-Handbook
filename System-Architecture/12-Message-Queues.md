# Chapter 12 — Message Queues

> **Part II — System Architecture**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 12.1 What is a Message Queue?

A **message queue** is a durable, temporary storage buffer that sits between a producer and a consumer. The producer puts messages in. The consumer takes messages out. The queue holds them safely in between — even if the consumer is offline, slow, or busy.

```
┌──────────────────────────────────────────────────────────────┐
│                 What a Message Queue Does                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  WITHOUT a queue:                                            │
│                                                              │
│  Producer ──────────────────────────────────► Consumer       │
│  (Registration)    direct call               (Biometric)    │
│                                                              │
│  Problem: If Biometric is down → Registration fails         │
│  Problem: If 1000 packets arrive → Biometric overwhelmed    │
│                                                              │
│  WITH a queue:                                               │
│                                                              │
│  Producer ──► [■■■■■■■■■■■■] ──► Consumer                   │
│  (Registration)  Queue          (Biometric)                 │
│                  (buffer)                                    │
│                                                              │
│  Benefit: Biometric down → messages wait safely in queue    │
│  Benefit: 1000 packets → queue absorbs burst, consumer      │
│           processes at its own pace                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

The queue provides three fundamental guarantees:
- **Durability** — messages are not lost even if the consumer crashes
- **Decoupling** — producer and consumer are independent
- **Load leveling** — burst traffic is absorbed, smoothed out for the consumer

---

## 12.2 Amazon SQS — The AWS Queue Service

**Amazon Simple Queue Service (SQS)** is AWS's fully managed message queuing service. In NBIS, SQS is the backbone of the entire enrollment write path.

```
┌──────────────────────────────────────────────────────────────┐
│              Why SQS for NBIS                                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ Fully managed — no servers to provision or maintain     │
│  ✅ Unlimited throughput (Standard) / 3k msg/s (FIFO)       │
│  ✅ Messages replicated across 3 AZs — no data loss         │
│  ✅ Encryption at rest (KMS) and in transit (TLS)           │
│  ✅ Dead Letter Queue support built-in                      │
│  ✅ Integrates natively with Lambda, ECS, EventBridge       │
│  ✅ Pay per message — no idle cost                          │
│  ✅ Message retention up to 14 days                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 12.3 SQS Core Concepts

### 12.3.1 Message

A **message** is the unit of work sent through the queue. In NBIS every message is a JSON event:

```
┌──────────────────────────────────────────────────────────────┐
│                   SQS Message Structure                      │
└──────────────────────────────────────────────────────────────┘
{
  // SQS envelope (added by SQS automatically)
  "MessageId":         "abc-123-def-456",
  "ReceiptHandle":     "AQEBwJ...long...string",  ← used to delete
  "MD5OfBody":         "a1b2c3d4...",
  "Attributes": {
    "SentTimestamp":         "1705312800000",
    "ApproximateReceiveCount": "1",               ← retry counter
    "ApproximateFirstReceiveTimestamp": "1705312801000"
  },

  // Body (our event payload — max 256 KB)
  "Body": "{
    \"eventId\":       \"EVT-20250115-abc123\",
    \"eventType\":     \"ENROLLMENT_RECEIVED\",
    \"eventVersion\":  \"1.0\",
    \"correlationId\": \"PKT-20250115-001\",
    \"timestamp\":     \"2025-01-15T10:30:00Z\",
    \"payload\": {
      \"packetId\":  \"PKT-20250115-001\",
      \"centerId\":  \"CENTER-001\",
      \"operatorId\": \"OFR-007\"
    }
  }"
}
```

**Message size limit: 256 KB.** For larger payloads (biometric templates), use the **S3 pointer pattern**:

```
// Instead of putting biometric data in the message:
❌ { "biometricData": "base64...5MB..." }

// Put the data in S3 and reference it:
✅ { "biometricRef": "s3://nbis-packets/PKT-001/biometrics.enc" }

Consumer reads message → fetches actual data from S3
→ No size limit, no extra SQS cost for large payloads
```

---

### 12.3.2 Send, Receive, Delete — The Core Cycle

```
┌──────────────────────────────────────────────────────────────┐
│              SQS Message Lifecycle                           │
└──────────────────────────────────────────────────────────────┘

PRODUCER:
  sqs.send_message(
    QueueUrl    = "https://sqs.region.amazonaws.com/.../queue",
    MessageBody = json(event),
    MessageAttributes = {
      "eventType": { "DataType": "String",
                     "StringValue": "ENROLLMENT_RECEIVED" }
    }
  )
  → Message stored in SQS (3 AZ replicas)
  → Producer returns immediately

CONSUMER (polling loop):
  messages = sqs.receive_message(
    QueueUrl            = queue_url,
    MaxNumberOfMessages = 10,          ← batch up to 10
    WaitTimeSeconds     = 20,          ← long polling
    VisibilityTimeout   = 120          ← 2 min to process
  )

  for message in messages:
    try:
      process(message.body)
      sqs.delete_message(             ← only delete on SUCCESS
        QueueUrl      = queue_url,
        ReceiptHandle = message.receipt_handle
      )
    except Exception as e:
      log_error(e)
      // Do NOT delete → message becomes visible again after
      // VisibilityTimeout and will be retried
```

---

### 12.3.3 Visibility Timeout — The Core Safety Mechanism

The **visibility timeout** is the period during which a message, after being received by a consumer, is hidden from other consumers. It is the mechanism that prevents two consumers from processing the same message simultaneously.

```
┌──────────────────────────────────────────────────────────────┐
│              Visibility Timeout — Full Timeline              │
└──────────────────────────────────────────────────────────────┘

T=0s    Message arrives in queue
        Status: VISIBLE (available for pickup)
             │
             ▼
T=5s    Consumer A picks up message
        Status: INVISIBLE (visibility timeout starts: 120s)
             │
        Consumer A processing...
             │
T=80s   Processing SUCCEEDS
        Consumer A calls delete_message()
        Status: DELETED (gone from queue forever) ✅
             │
─────────────────────────────────────────────────────────────
        ALTERNATIVE — processing FAILS or takes too long:
             │
T=125s  Visibility timeout EXPIRES (no delete was called)
        Status: VISIBLE again (available for pickup)
             │
             ▼
T=126s  Consumer B picks up same message (retry attempt 2)
        ApproximateReceiveCount = 2
             │
        ...
             │
T=???   After maxReceiveCount (e.g. 3):
        Message moved to DLQ automatically
        Status: IN DLQ (needs manual investigation)
```

### Visibility timeout tuning per NBIS queue

```
┌──────────────────────────────────────────────────────────────┐
│          Visibility Timeout Tuning Guide                     │
├──────────────────────────────────┬───────────────────────────┤
│ Queue                            │ Timeout   │ Reason        │
├──────────────────────────────────┼───────────┼───────────────┤
│ nbis-enrollment-received         │ 60s       │ Simple packet │
│                                  │           │ validation    │
├──────────────────────────────────┼───────────┼───────────────┤
│ nbis-biometric-processing        │ 120s      │ Quality check │
│                                  │           │ + extraction  │
├──────────────────────────────────┼───────────┼───────────────┤
│ nbis-abis-requests               │ 600s      │ ABIS 1:N      │
│                                  │           │ search slow   │
├──────────────────────────────────┼───────────┼───────────────┤
│ nbis-abis-results                │ 30s       │ Just routing  │
│                                  │           │ result        │
├──────────────────────────────────┼───────────┼───────────────┤
│ nbis-uin-assignment.fifo         │ 30s       │ Fast DB write │
├──────────────────────────────────┼───────────┼───────────────┤
│ nbis-credential-issuance         │ 120s      │ Sign + store  │
│                                  │           │ VC artifact   │
├──────────────────────────────────┼───────────┼───────────────┤
│ nbis-notifications               │ 60s       │ SMS delivery  │
├──────────────────────────────────┼───────────┼───────────────┤
│ nbis-civil-registry-events       │ 60s       │ Revocation    │
│                                  │           │ trigger       │
└──────────────────────────────────┴───────────┴───────────────┘

Rule: Timeout = (expected processing time) × 3
      Never set it lower than your average processing time
      or you will get duplicate processing.
```

### Extending visibility timeout for long jobs

For the ABIS queue where processing can take longer than expected:

```
// If processing is taking longer than expected,
// extend the timeout BEFORE it expires:

function process_abis_job(message):
  job = submit_to_abis(message.payload)

  while not job.complete:
    sleep(30 seconds)

    // If we have less than 60s remaining, extend
    if time_remaining(message) < 60:
      sqs.change_message_visibility(
        QueueUrl        = abis_queue_url,
        ReceiptHandle   = message.receipt_handle,
        VisibilityTimeout = 300  // extend by 5 more minutes
      )

  result = job.get_result()
  process_result(result)
  sqs.delete_message(message.receipt_handle)
```

---

## 12.4 SQS Standard Queue

**Standard queues** are the default SQS queue type — optimized for maximum throughput.

```
┌──────────────────────────────────────────────────────────────┐
│              SQS Standard Queue Properties                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Throughput:       Nearly unlimited (thousands/second)       │
│  Delivery:         At-least-once (may deliver more than once)│
│  Ordering:         Best-effort (not guaranteed)              │
│  Deduplication:    Not built-in (consumer must handle)       │
│  Pricing:          $0.40 per 1 million requests             │
│                                                              │
│  Best for:                                                   │
│  ├── High-volume pipelines where order does not matter      │
│  ├── Stages where idempotency handles duplicates            │
│  ├── Notification delivery (order irrelevant)               │
│  └── ABIS job submission (each job is independent)          │
│                                                              │
│  NBIS queues using Standard:                                 │
│  ├── nbis-enrollment-received                               │
│  ├── nbis-biometric-processing                              │
│  ├── nbis-abis-requests                                     │
│  ├── nbis-abis-results                                      │
│  ├── nbis-credential-issuance                               │
│  └── nbis-notifications                                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 12.5 SQS FIFO Queue

**FIFO (First-In, First-Out) queues** guarantee strict message ordering and exactly-once processing. Used for NBIS stages where sequence matters.

```
┌──────────────────────────────────────────────────────────────┐
│              SQS FIFO Queue Properties                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Throughput:       3,000 msg/s with batching                 │
│                    300 msg/s without batching                │
│  Delivery:         Exactly-once (built-in deduplication)    │
│  Ordering:         Strict FIFO per MessageGroupId           │
│  Deduplication:    Built-in (MessageDeduplicationId)        │
│  Pricing:          $0.50 per 1 million requests             │
│  Naming:           Queue name MUST end in .fifo             │
│                                                              │
│  Best for:                                                   │
│  ├── UIN assignment (one UIN per packet — order critical)   │
│  ├── Lifecycle state transitions (must be sequential)       │
│  └── Civil registry events (birth before enrollment)        │
│                                                              │
│  NBIS queues using FIFO:                                     │
│  ├── nbis-uin-assignment.fifo                               │
│  └── nbis-civil-registry-events.fifo                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### FIFO — MessageGroupId and MessageDeduplicationId

```
┌──────────────────────────────────────────────────────────────┐
│              FIFO Key Concepts                               │
└──────────────────────────────────────────────────────────────┘

MessageGroupId:
  Groups related messages together for ordered processing.
  Within a group, messages are processed strictly in order.
  Different groups can be processed in parallel.

  For UIN assignment:
  MessageGroupId = packetId
  → All events for PKT-001 processed in order
  → PKT-002 processed in parallel (different group)

  sqs.send_message(
    QueueUrl                = "nbis-uin-assignment.fifo",
    MessageBody             = json(event),
    MessageGroupId          = event.packetId,
    MessageDeduplicationId  = event.eventId
  )

MessageDeduplicationId:
  SQS uses this to detect and discard duplicate sends.
  If you send the same MessageDeduplicationId twice within
  5 minutes, the second send is silently ignored.

  For NBIS: use eventId as MessageDeduplicationId
  → If Registration Service sends the same event twice
    (network retry), SQS only delivers it once

  This gives you exactly-once delivery at the queue level.
  (Still implement idempotency in consumer for safety.)
```

---

## 12.6 Standard vs FIFO — Decision Guide

```
┌──────────────────────────────────────────────────────────────┐
│           Standard vs FIFO Decision Tree                     │
└──────────────────────────────────────────────────────────────┘

Does message ORDER matter?
├── NO → Use Standard Queue
│        (faster, cheaper, higher throughput)
│
└── YES → Can you handle duplicates with idempotency?
          ├── YES → Standard Queue + idempotency is fine
          │         (usually preferred for throughput)
          │
          └── NO → Use FIFO Queue
                   (strict ordering + exactly-once built in)

Does THROUGHPUT > 3,000 msg/s matter?
├── YES → Standard Queue only (FIFO cannot handle it)
└── NO  → Either queue type works

Quick reference for NBIS:
┌──────────────────────────────┬───────────────┐
│ Queue                        │ Type          │
├──────────────────────────────┼───────────────┤
│ enrollment-received          │ Standard      │
│ biometric-processing         │ Standard      │
│ abis-requests                │ Standard      │
│ abis-results                 │ Standard      │
│ credential-issuance          │ Standard      │
│ notifications                │ Standard      │
│ dedup-review                 │ Standard      │
│ uin-assignment               │ FIFO ✅       │
│ civil-registry-events        │ FIFO ✅       │
└──────────────────────────────┴───────────────┘
```

---

## 12.7 Dead Letter Queue — Complete Design

```
┌──────────────────────────────────────────────────────────────┐
│              Dead Letter Queue Architecture                  │
└──────────────────────────────────────────────────────────────┘

SETUP:
Every main queue has a paired DLQ.
DLQ is always Standard type (even if main is FIFO).

Main queue settings:
  maxReceiveCount: 3          ← move to DLQ after 3 failures
  redrivePolicy: {
    deadLetterTargetArn: "arn:aws:sqs:...:nbis-enrollment-dlq",
    maxReceiveCount: 3
  }

DLQ settings:
  messageRetentionPeriod: 1209600  ← 14 days (max)
  (gives operations team time to investigate and replay)

ALARM (CloudWatch):
  MetricName:  ApproximateNumberOfMessagesVisible
  Namespace:   AWS/SQS
  Dimensions:  QueueName = nbis-enrollment-received-dlq
  Threshold:   >= 1
  Period:      60 seconds
  Action:      SNS → email + PagerDuty
  Priority:    P1 (enrollment DLQ = citizen without identity)

DLQ CONTENTS ANALYSIS:
When a message lands in DLQ, investigate:

  Step 1: Read the message
  Step 2: Check ApproximateReceiveCount (how many times retried)
  Step 3: Check error logs with correlationId
  Step 4: Determine failure type:
    ├── Transient (ABIS was down) → fix ABIS → replay
    ├── Data error (malformed packet) → fix data → replay
    └── Code bug → fix code → replay

DLQ REPLAY (after fix):
  // Move messages back to main queue for reprocessing
  aws sqs start-message-move-task \
    --source-queue-url https://sqs.../nbis-enrollment-dlq \
    --destination-queue-url https://sqs.../nbis-enrollment-received
```

---

## 12.8 Message Retention and Throughput

```
┌──────────────────────────────────────────────────────────────┐
│           SQS Configuration Reference                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Message retention period:                                   │
│  ├── Default: 4 days                                        │
│  ├── Minimum: 60 seconds                                    │
│  ├── Maximum: 14 days                                       │
│  └── NBIS setting: 7 days (main), 14 days (DLQ)            │
│                                                              │
│  Message size:                                               │
│  ├── Maximum: 256 KB per message                            │
│  └── For larger: use S3 pointer pattern                     │
│                                                              │
│  Batch operations:                                           │
│  ├── Send up to 10 messages in one API call                 │
│  ├── Receive up to 10 messages in one API call              │
│  └── Delete up to 10 messages in one API call               │
│  → Always batch for cost and efficiency                     │
│                                                              │
│  Long polling:                                               │
│  ├── WaitTimeSeconds: 0–20 seconds                         │
│  └── NBIS setting: 20 seconds (always max)                 │
│                                                              │
│  In-flight messages (Standard):                              │
│  ├── Maximum: 120,000 per queue                             │
│  └── If hit: consumer must be scaled up                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 12.9 Lambda Trigger vs Polling Consumer

SQS consumers can be implemented in two ways — polling (ECS service) or Lambda trigger:

```
┌──────────────────────────────────────────────────────────────┐
│         Lambda Trigger vs Polling — When to Use Which        │
├──────────────────────────┬───────────────────────────────────┤
│ Lambda Trigger           │ ECS Polling Consumer             │
├──────────────────────────┼───────────────────────────────────┤
│ AWS manages polling      │ Service manages own polling loop  │
│ Auto-scales to queue     │ Manual scaling config             │
│ depth                    │ (or auto-scaling policy)          │
│                          │                                   │
│ Max processing time:     │ No time limit                    │
│ 15 minutes               │                                   │
│                          │                                   │
│ Best for:                │ Best for:                        │
│ ├── UIN assignment       │ ├── ABIS jobs (> 15 min)         │
│ ├── Notifications        │ ├── Biometric extraction         │
│ └── Civil registry       │ └── Long-running processing      │
│     events               │                                   │
│                          │                                   │
│ AWS handles:             │ You handle:                      │
│ ├── Polling              │ ├── Polling loop                 │
│ ├── Scaling              │ ├── Scaling policy               │
│ ├── Concurrency          │ ├── Error handling               │
│ └── Error → DLQ          │ └── DLQ retry logic              │
└──────────────────────────┴───────────────────────────────────┘
```

### Lambda trigger configuration

```
// Lambda triggered by SQS — AWS handles everything
{
  "EventSourceArn": "arn:aws:sqs:...:nbis-uin-assignment.fifo",
  "BatchSize": 1,              // Process 1 UIN at a time (safe)
  "FunctionResponseTypes": ["ReportBatchItemFailures"],
  // ↑ Allows partial batch failure — successful items deleted,
  //   failed items returned to queue for retry
  "MaximumConcurrency": 10     // Max 10 parallel Lambda instances
}
```

---

## 12.10 Queue Encryption

All NBIS queues must be encrypted — messages contain enrollment data, UINs, and citizen references:

```
┌──────────────────────────────────────────────────────────────┐
│              SQS Encryption Configuration                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Encryption at rest:                                         │
│  KmsMasterKeyId: "arn:aws:kms:...:nbis-sqs-key"            │
│  (SSE-KMS — server-side encryption with customer key)       │
│                                                              │
│  Encryption in transit:                                      │
│  All SQS API calls use HTTPS (enforced by IAM policy)       │
│                                                              │
│  IAM policy to enforce TLS:                                  │
│  {                                                           │
│    "Effect": "Deny",                                         │
│    "Principal": "*",                                         │
│    "Action": "sqs:*",                                        │
│    "Resource": "arn:aws:sqs:...:nbis-*",                    │
│    "Condition": {                                            │
│      "Bool": { "aws:SecureTransport": "false" }             │
│    }                                                         │
│  }                                                           │
│  → Any non-HTTPS call to any NBIS queue is denied           │
│                                                              │
│  KMS key rotation: annual (automatic)                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 12.11 Access Control — Queue IAM Policies

Each queue has a resource policy controlling which services can send and receive:

```
┌──────────────────────────────────────────────────────────────┐
│         Queue-Level Access Control                           │
└──────────────────────────────────────────────────────────────┘

nbis-enrollment-received (only Registration Service can send):
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123:role/nbis-registration-service"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:...:nbis-enrollment-received"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123:role/nbis-biometric-service"
      },
      "Action": ["sqs:ReceiveMessage", "sqs:DeleteMessage",
                 "sqs:ChangeMessageVisibility"],
      "Resource": "arn:aws:sqs:...:nbis-enrollment-received"
    }
  ]
}

Principle: Only the producer can SendMessage.
           Only the consumer can ReceiveMessage + DeleteMessage.
           No other service has any access.
```

---

## 12.12 Monitoring and Observability

```
┌──────────────────────────────────────────────────────────────┐
│              SQS CloudWatch Metrics Reference                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Key metrics per queue:                                      │
│  ┌──────────────────────────────────┬──────────────────────┐ │
│  │ Metric                           │ Alarm threshold     │ │
│  ├──────────────────────────────────┼──────────────────────┤ │
│  │ ApproximateNumberOfMessages      │ > 1000 (backlog)    │ │
│  │ Visible                          │                     │ │
│  ├──────────────────────────────────┼──────────────────────┤ │
│  │ ApproximateAgeOfOldestMessage    │ > 600s (10 min)     │ │
│  ├──────────────────────────────────┼──────────────────────┤ │
│  │ NumberOfMessagesSent             │ (throughput monitor) │ │
│  ├──────────────────────────────────┼──────────────────────┤ │
│  │ NumberOfMessagesDeleted          │ (success monitor)   │ │
│  ├──────────────────────────────────┼──────────────────────┤ │
│  │ NumberOfMessagesNotVisible       │ (in-flight monitor) │ │
│  │ (in-flight)                      │                     │ │
│  ├──────────────────────────────────┼──────────────────────┤ │
│  │ DLQ:                             │                     │ │
│  │ ApproximateNumberOfMessages      │ >= 1 (P1 alert)     │ │
│  │ Visible                          │                     │ │
│  └──────────────────────────────────┴──────────────────────┘ │
│                                                              │
│  Dashboard: one CloudWatch dashboard per environment        │
│  ├── Queue depth graph (all queues)                         │
│  ├── Message age graph (all queues)                         │
│  ├── DLQ depth (red — should always be 0)                  │
│  └── Throughput (messages/min trend)                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 12.13 Batch Processing for Cost Efficiency

Processing messages one at a time is inefficient. NBIS uses batching wherever safe:

```
┌──────────────────────────────────────────────────────────────┐
│              Batching Strategy                               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Receive batch (always):                                     │
│  sqs.receive_message(MaxNumberOfMessages=10)                │
│  → One API call receives up to 10 messages                  │
│  → 10x fewer API calls = 10x cheaper                       │
│                                                              │
│  Process batch (where safe):                                 │
│  Notifications → process 10 at once (send 10 SMS)          │
│  Biometric quality → process 1 at a time (complex, risky)  │
│  UIN assignment → process 1 at a time (FIFO, critical)     │
│                                                              │
│  Delete batch (always):                                      │
│  sqs.delete_message_batch(Entries=[...10 messages...])      │
│  → One API call to delete all successful messages           │
│                                                              │
│  Partial batch failure (Lambda):                             │
│  If 8 of 10 messages succeed and 2 fail:                    │
│  → Return failed message IDs in batchItemFailures           │
│  → AWS deletes the 8 successful ones                        │
│  → The 2 failed ones stay in queue for retry                │
│  → Do NOT manually delete failed messages                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 12.14 Complete NBIS Queue Architecture

```
┌──────────────────────────────────────────────────────────────┐
│           Complete NBIS SQS Architecture                     │
└──────────────────────────────────────────────────────────────┘

ENROLLMENT WRITE PATH:

Registration Service
  │ SendMessage (Standard)
  ▼
nbis-enrollment-received ──────────► nbis-enrollment-received-dlq
  │ ReceiveMessage (ECS Biometric)         (CloudWatch Alarm: P1)
  ▼
Biometric Service (ECS Fargate)
  │ SendMessage (Standard)
  ▼
nbis-biometric-processing ─────────► nbis-biometric-dlq
  │ ReceiveMessage (Lambda)                (CloudWatch Alarm: P1)
  ▼
Deduplication Service (Lambda)
  │ SendMessage (Standard)
  ▼
nbis-abis-requests ────────────────► nbis-abis-dlq
  │ ReceiveMessage (ABIS)                  (CloudWatch Alarm: P1)
  ▼
[ABIS — External Vendor]
  │ SendMessage (Standard)
  ▼
nbis-abis-results ─────────────────► nbis-abis-results-dlq
  │ ReceiveMessage (Lambda)               (CloudWatch Alarm: P1)
  ▼
Deduplication Result Processor
  │
  ├── FLAGGED → nbis-dedup-review (Standard)
  │              → Supervisor Portal picks up
  │
  └── CLEARED → nbis-uin-assignment.fifo ► nbis-uin-dlq
                  │ ReceiveMessage (Lambda)   (P1 alarm)
                  ▼
                UIN Service
                  │ PutEvents → EventBridge
                  ▼
         EventBridge (nbis-internal)
                  │
         ┌────────┼──────────┐
         ▼        ▼          ▼
   nbis-cred  nbis-notif  nbis-analytics
   (Standard) (Standard)  (Standard)
         │        │
         ▼        ▼
   Credential  Notification
   Service     Service
   → Smart     → SMS
     card VC   → Email

EXTERNAL EVENTS PATH:

Civil Registry
  │ PutEvents
  ▼
EventBridge (nbis-external)
  │ Rule: CITIZEN_DECEASED
  ▼
nbis-civil-registry-events.fifo ──► nbis-civil-registry-dlq
  │ ReceiveMessage (Lambda)             (CloudWatch Alarm: P1)
  ▼
Revocation Handler
  → Suspend auth immediately
  → Trigger full revocation workflow
```

---

## 12.15 Cost Estimation

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS SQS Cost Estimation                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Scenario: 10,000 enrollments / day                         │
│  Each enrollment generates ~8 queue messages                │
│  = 80,000 messages/day = 2.4M messages/month               │
│                                                              │
│  Verification: 1,000,000 authentications/day                │
│  (verification uses Lambda → DynamoDB, NOT SQS)             │
│  → SQS cost is enrollment-only                             │
│                                                              │
│  Cost:                                                       │
│  Standard queue: $0.40 per 1M requests                     │
│  2.4M messages × $0.40/1M = ~$1/month                     │
│                                                              │
│  Plus: KMS calls for encryption ~$0.30/million             │
│  Total SQS + KMS: ~$2–3/month for 10k enrollments/day     │
│                                                              │
│  SQS is effectively free at NBIS enrollment volumes.        │
│  The expensive part is ABIS (external vendor pricing).      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 12.16 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Message Queue Reference                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Default MOSIP deployment uses Apache Kafka:                │
│  ├── Kafka topics map 1:1 to our SQS queues                │
│  ├── Consumer groups = our ECS consumer services           │
│  └── Kafka retention = our SQS message retention           │
│                                                              │
│  MOSIP Kafka topic → NBIS SQS equivalent:                   │
│  ┌─────────────────────────────────┬──────────────────────┐  │
│  │ MOSIP Kafka Topic               │ NBIS SQS Queue       │  │
│  ├─────────────────────────────────┼──────────────────────┤  │
│  │ mosip.regproc.packet.receiver   │ enrollment-received  │  │
│  │ mosip.regproc.packet.validator  │ biometric-processing │  │
│  │ mosip.regproc.abis.handler      │ abis-requests        │  │
│  │ mosip.regproc.uin.generator     │ uin-assignment       │  │
│  │ mosip.regproc.notification      │ notifications        │  │
│  └─────────────────────────────────┴──────────────────────┘  │
│                                                              │
│  For AWS deployments:                                        │
│  Kafka → replaced with Amazon MSK (managed Kafka)           │
│  or Amazon SQS (if Kafka features not needed)               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 12.17 Key Terms

| Term | Definition |
|------|-----------|
| **Message queue** | Durable buffer between producer and consumer — decouples and smooths load |
| **SQS** | Amazon Simple Queue Service — AWS managed message queue |
| **Standard queue** | SQS type: unlimited throughput, at-least-once delivery, best-effort order |
| **FIFO queue** | SQS type: strict ordering, exactly-once delivery, 3,000 msg/s max |
| **MessageGroupId** | FIFO key that groups messages for ordered processing within the group |
| **MessageDeduplicationId** | FIFO key that prevents duplicate message delivery within 5 minutes |
| **Visibility timeout** | Period a received message is hidden from other consumers |
| **ReceiptHandle** | Token needed to delete or extend visibility of a received message |
| **Dead Letter Queue** | Destination for messages that fail after maxReceiveCount retries |
| **maxReceiveCount** | How many times a message can be received before going to DLQ |
| **Long polling** | Consumer waits up to 20s for messages — reduces empty API calls |
| **S3 pointer pattern** | Store large payload in S3, put only the S3 reference in SQS message |
| **Batch processing** | Receive / delete up to 10 messages per API call for efficiency |
| **Partial batch failure** | Lambda feature — failed messages retry while successful ones are deleted |
| **SSE-KMS** | Server-side encryption with KMS key — protects messages at rest |
| **Backpressure** | Queue depth rising because consumer cannot keep up with producer |
| **Redrive policy** | SQS config that defines DLQ target and maxReceiveCount |
| **In-flight message** | Message received but not yet deleted — invisible to other consumers |

---

## 12.18 Key Takeaways

- **Message queues decouple producer from consumer** — the producer never waits for the consumer. They operate independently, at their own speeds.
- **Visibility timeout is the safety mechanism** — set it longer than your expected processing time or you will get duplicate processing. Tune per queue, not globally.
- **Always use long polling (WaitTimeSeconds=20)** — short polling wastes API calls and money. There is no downside to long polling in NBIS.
- **Standard queue + idempotency is usually the right choice** — FIFO's exactly-once and ordering come at a throughput cost. Use FIFO only where ordering genuinely matters (UIN assignment, civil registry events).
- **Every queue must have a DLQ with a P1 alarm** — a failed enrollment packet is a citizen without an identity. Never let a message be silently dropped.
- **Batch everything** — receive 10, process, delete 10. One API call instead of ten. SQS pricing is per request, not per message.
- **Use the S3 pointer pattern for large payloads** — biometric templates are megabytes. SQS messages are 256 KB max. Never try to embed templates in messages.
- **Queue IAM policies enforce separation** — only the Registration Service can send to the enrollment queue. Only the Biometric Service can receive from it. No other service touches it.

---

## 12.19 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 13 | Service Discovery — Cloud Map, internal ALB, how services find each other |
| Chapter 14 | Caching — ElastiCache Redis for OTP, sessions, and hot-path optimization |
| Chapter 15 | High Availability — Multi-AZ, health checks, failover strategies |

---

*Chapter 12 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
