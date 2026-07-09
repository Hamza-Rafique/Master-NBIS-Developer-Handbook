# Chapter 13 — Service Discovery

> **Part II — System Architecture**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 13.1 What is Service Discovery?

In a microservices architecture, services need to call each other. But in a dynamic containerized environment, services start, stop, crash, scale up, and scale down constantly. **IP addresses change. Ports change. Container instances come and go.**

**Service discovery** is the mechanism by which a service finds the current network location (IP + port) of another service it needs to call — automatically, without hardcoding addresses.

```
┌──────────────────────────────────────────────────────────────┐
│              The Problem Service Discovery Solves            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  WITHOUT service discovery:                                  │
│                                                              │
│  Auth Service config:                                        │
│  biometric_service_url = "http://10.0.1.45:8080"           │
│                                                              │
│  Problems:                                                   │
│  ├── Biometric Service container crashes                    │
│  ├── ECS starts a new container at 10.0.1.67:8080          │
│  ├── Auth Service still points to dead 10.0.1.45           │
│  └── Auth Service breaks → citizens cannot authenticate     │
│                                                              │
│  Also:                                                       │
│  ├── Biometric Service scales to 5 instances               │
│  ├── Auth Service only knows about 1 of them               │
│  └── 4 instances sit idle while 1 is overwhelmed           │
│                                                              │
│  WITH service discovery:                                     │
│                                                              │
│  Auth Service config:                                        │
│  biometric_service_url = "http://biometric.nbis.local"     │
│                                                              │
│  Benefits:                                                   │
│  ├── DNS name always resolves to healthy instances          │
│  ├── New containers register automatically                  │
│  ├── Dead containers deregister automatically               │
│  └── Load balanced across all healthy instances            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 13.2 How Service Discovery Works

Service discovery has two components:

```
┌──────────────────────────────────────────────────────────────┐
│          Two Components of Service Discovery                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. SERVICE REGISTRY                                         │
│     A database of all running service instances             │
│     ├── Service name                                        │
│     ├── IP address + port                                   │
│     ├── Health status (healthy / unhealthy)                 │
│     └── Metadata (version, environment, region)             │
│                                                              │
│  2. SERVICE LOOKUP                                           │
│     How a caller finds a service's address                  │
│     ├── Client-side: caller queries registry directly       │
│     └── Server-side: load balancer or DNS handles it        │
│                                                              │
│  In AWS NBIS:                                                │
│  ├── Registry: AWS Cloud Map OR internal ALB target groups  │
│  └── Lookup: DNS (Route 53 private hosted zone) OR ALB DNS  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 13.3 Service Discovery Patterns

There are three approaches used in NBIS:

```
┌──────────────────────────────────────────────────────────────┐
│         Service Discovery Pattern Comparison                 │
├──────────────────────────┬───────────────────────────────────┤
│ Pattern                  │ How it works                     │
├──────────────────────────┼───────────────────────────────────┤
│ 1. Internal ALB          │ Services talk to a stable DNS    │
│    (Application Load     │ name of an ALB. ALB routes to   │
│    Balancer)             │ healthy ECS tasks automatically. │
│                          │                                  │
│                          │ NBIS primary pattern.            │
├──────────────────────────┼───────────────────────────────────┤
│ 2. AWS Cloud Map         │ Services register themselves via │
│    (DNS-based)           │ ECS service discovery. Other     │
│                          │ services look up by DNS name.    │
│                          │                                  │
│                          │ NBIS secondary pattern for       │
│                          │ lightweight internal calls.      │
├──────────────────────────┼───────────────────────────────────┤
│ 3. API Gateway +         │ External callers → API Gateway   │
│    VPC Link              │ → Internal ALB → ECS service.   │
│    (external entry)      │                                  │
│                          │ NBIS pattern for all external    │
│                          │ traffic (Chapter 8).             │
└──────────────────────────┴───────────────────────────────────┘
```

---

## 13.4 Pattern 1 — Internal Application Load Balancer

The **primary service discovery pattern in NBIS**. Each microservice gets its own internal ALB. Other services talk to the ALB's stable DNS name — not to individual ECS tasks.

```
┌──────────────────────────────────────────────────────────────┐
│              Internal ALB Service Discovery                  │
└──────────────────────────────────────────────────────────────┘

SETUP:
Each microservice has:
  ├── ECS Service (manages task instances)
  ├── Internal ALB (load balancer — not public)
  ├── Target Group (ECS tasks registered here)
  └── DNS name: service-name.nbis.internal

REGISTRATION (automatic):
When ECS starts a new task:
  1. Container starts
  2. Health check endpoint responds: GET /actuator/health → 200
  3. ECS registers task IP:port in ALB Target Group
  4. ALB starts routing traffic to it

DEREGISTRATION (automatic):
When ECS task crashes or is stopped:
  1. Health check fails
  2. ALB removes task from Target Group
  3. No more traffic routed to dead task
  4. ECS starts replacement task → registered automatically

CALLER:
Auth Service calling Biometric Service:

  // Hard-coded stable DNS — never changes
  BIOMETRIC_URL = "http://biometric-svc.nbis.internal/v1"

  response = http.post(
    url  = BIOMETRIC_URL + "/quality-check",
    body = { templateData: "..." },
    timeout = 10_seconds
  )

  // No awareness of IPs, ports, or how many instances exist
  // ALB handles routing + load balancing
```

### Internal ALB architecture per service

```
┌──────────────────────────────────────────────────────────────┐
│          Internal ALB Architecture — Auth Service            │
└──────────────────────────────────────────────────────────────┘

Callers (Biometric Svc, eKYC Svc, API Gateway VPC Link)
         │
         │ http://auth-svc.nbis.internal
         ▼
┌─────────────────────────────────────┐
│  Internal ALB                       │
│  auth-svc-alb.nbis.internal        │
│  (private subnet — not internet     │
│   accessible)                       │
└──────────────┬──────────────────────┘
               │ routes to healthy tasks
         ┌─────┼─────┐
         ▼     ▼     ▼
      Task 1  Task 2  Task 3
      AZ-a    AZ-b    AZ-c
      10.0.1.10 10.0.2.10 10.0.3.10
      :8080   :8080   :8080

      All running Auth Service v2.1.0
      All passing health checks
      ALB distributes requests round-robin

When Task 2 crashes:
      ALB detects health check failure
      Removes 10.0.2.10 from rotation
      Traffic split between Task 1 and Task 3
      ECS launches Task 4 in AZ-b
      Task 4 passes health check → added to rotation
      All in ~30 seconds, zero manual intervention
```

---

## 13.5 Pattern 2 — AWS Cloud Map

**AWS Cloud Map** is a fully managed service registry. ECS services can register themselves automatically with Cloud Map, making them discoverable by name via DNS.

```
┌──────────────────────────────────────────────────────────────┐
│              AWS Cloud Map Architecture                      │
└──────────────────────────────────────────────────────────────┘

SETUP:
1. Create Cloud Map namespace:
   nbis.local (private DNS namespace — only inside VPC)

2. Create service in namespace:
   biometric-svc.nbis.local

3. ECS Service → enable Service Discovery:
   {
     "serviceRegistries": [{
       "registryArn": "arn:aws:servicediscovery:...:service/srv-xxx"
     }]
   }

REGISTRATION (automatic):
ECS starts task → registers in Cloud Map:
  biometric-svc.nbis.local → 10.0.1.15

ECS starts 2nd task → Cloud Map adds record:
  biometric-svc.nbis.local → 10.0.1.15
  biometric-svc.nbis.local → 10.0.2.22  (multi-value DNS)

LOOKUP (by caller):
  Auth Service:
  dns.resolve("biometric-svc.nbis.local")
  → [10.0.1.15, 10.0.2.22]
  → pick one (client-side load balancing)

HEALTH CHECK:
  Cloud Map runs health checks on registered instances
  Unhealthy instances removed from DNS responses
```

### Cloud Map vs Internal ALB — when to use which

```
┌──────────────────────────────────────────────────────────────┐
│         Cloud Map vs Internal ALB Decision Guide             │
├──────────────────────────┬───────────────────────────────────┤
│ Use Internal ALB when:   │ Use Cloud Map when:              │
├──────────────────────────┼───────────────────────────────────┤
│ Need server-side load    │ Client-side load balancing OK    │
│ balancing                │                                  │
│                          │                                  │
│ Need path-based routing  │ Simple DNS lookup sufficient     │
│ (/v1 vs /v2)            │                                   │
│                          │                                  │
│ Need SSL termination     │ HTTP between services only       │
│ at the load balancer     │                                  │
│                          │                                  │
│ High traffic (millions   │ Low/medium traffic internal      │
│ of requests)             │ calls                           │
│                          │                                  │
│ API Gateway needs to     │ Service mesh / sidecar patterns  │
│ route to it (VPC Link)   │                                  │
│                          │                                  │
│ NBIS: Auth, eKYC,        │ NBIS: lightweight internal       │
│ Verification (hot path)  │ admin calls, health queries      │
└──────────────────────────┴───────────────────────────────────┘
```

---

## 13.6 Health Checks — The Foundation of Discovery

Service discovery is only useful if it knows which instances are **actually healthy**. Health checks are how the system knows.

```
┌──────────────────────────────────────────────────────────────┐
│              Health Check Design in NBIS                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Every NBIS microservice exposes:                           │
│  GET /actuator/health                                        │
│  (Spring Boot Actuator — standard endpoint)                 │
│                                                              │
│  Response when healthy:                                      │
│  HTTP 200                                                    │
│  {                                                           │
│    "status": "UP",                                           │
│    "components": {                                           │
│      "db":    { "status": "UP" },                           │
│      "redis": { "status": "UP" },                           │
│      "disk":  { "status": "UP" }                            │
│    }                                                         │
│  }                                                           │
│                                                              │
│  Response when unhealthy:                                    │
│  HTTP 503                                                    │
│  {                                                           │
│    "status": "DOWN",                                         │
│    "components": {                                           │
│      "db": { "status": "DOWN",                              │
│              "details": { "error": "Connection refused" }}  │
│    }                                                         │
│  }                                                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Three levels of health check

```
┌──────────────────────────────────────────────────────────────┐
│          Health Check Types in NBIS                          │
├──────────────────────┬───────────────────────────────────────┤
│ Type                 │ Purpose                              │
├──────────────────────┼───────────────────────────────────────┤
│ Liveness check       │ "Is the process alive?"              │
│ /actuator/health/    │ If dead → restart container          │
│ liveness             │ Check: is the JVM running?           │
│                      │ ECS: container health check          │
├──────────────────────┼───────────────────────────────────────┤
│ Readiness check      │ "Is the service ready for traffic?"  │
│ /actuator/health/    │ If not ready → remove from ALB       │
│ readiness            │ Check: DB connected? Cache warm?     │
│                      │ ALB: target group health check       │
├──────────────────────┼───────────────────────────────────────┤
│ Deep health check    │ "Are all dependencies healthy?"      │
│ /actuator/health     │ Check: DynamoDB, S3, Redis,         │
│                      │        downstream services           │
│                      │ Monitoring + alerting only           │
│                      │ Do NOT use for ALB (too strict)      │
└──────────────────────┴───────────────────────────────────────┘

Rule: ALB uses readiness check only.
      A service can be up but its dependency is slow.
      Do not take the service out of rotation for
      a downstream issue it cannot control.
```

### Health check configuration (ALB Target Group)

```
┌──────────────────────────────────────────────────────────────┐
│          ALB Health Check Configuration                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Protocol:             HTTP                                  │
│  Path:                 /actuator/health/readiness            │
│  Port:                 8080 (service port)                   │
│  Healthy threshold:    2  (2 consecutive passes = healthy)  │
│  Unhealthy threshold:  3  (3 consecutive fails = unhealthy) │
│  Timeout:              5 seconds                            │
│  Interval:             10 seconds                           │
│  Success codes:        200                                   │
│                                                              │
│  Effect:                                                     │
│  New task starts → must pass 2 health checks (20s)          │
│  before getting traffic                                      │
│                                                              │
│  Task fails → must fail 3 checks (30s) before removed       │
│  (small grace period for transient failures)                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 13.7 Service Registration in ECS

When using ECS with an internal ALB, service registration is fully automatic — no manual configuration needed:

```
┌──────────────────────────────────────────────────────────────┐
│         ECS → ALB Auto-Registration Flow                     │
└──────────────────────────────────────────────────────────────┘

ECS Task Definition:
{
  "family": "nbis-auth-service",
  "containerDefinitions": [{
    "name": "auth-service",
    "image": "123456789.dkr.ecr.region.amazonaws.com/nbis-auth:v2.1",
    "portMappings": [{ "containerPort": 8080 }],
    "healthCheck": {
      "command": ["CMD-SHELL",
                  "curl -f http://localhost:8080/actuator/health/readiness || exit 1"],
      "interval": 10,
      "timeout": 5,
      "retries": 3,
      "startPeriod": 30    ← grace period on startup
    }
  }]
}

ECS Service Definition:
{
  "serviceName": "nbis-auth-service",
  "cluster": "nbis-production",
  "desiredCount": 3,
  "loadBalancers": [{
    "targetGroupArn": "arn:aws:elasticloadbalancing:...:targetgroup/auth-svc-tg",
    "containerName": "auth-service",
    "containerPort": 8080
  }],
  "deploymentConfiguration": {
    "minimumHealthyPercent": 100,   ← never drop below 3 healthy tasks
    "maximumPercent": 200           ← can have 6 tasks during deployment
  }
}

WHAT HAPPENS AUTOMATICALLY:
1. ECS launches task on EC2/Fargate node
2. Container starts (startPeriod: 30s grace)
3. Health check begins: GET /actuator/health/readiness
4. After 2 passes: ECS registers task IP:port in target group
5. ALB starts routing traffic to this task
6. Service is discoverable at auth-svc.nbis.internal
```

---

## 13.8 Service-to-Service Communication in NBIS

With service discovery in place, here is how each NBIS service finds and calls the others:

```
┌──────────────────────────────────────────────────────────────┐
│          NBIS Service-to-Service Call Map                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Who calls whom (synchronous REST calls):                   │
│                                                              │
│  API Gateway VPC Link                                        │
│    │                                                         │
│    ├──► auth-svc.nbis.internal         (Auth Service)       │
│    ├──► ekyc-svc.nbis.internal         (eKYC Service)       │
│    ├──► resident-svc.nbis.internal     (Resident Service)   │
│    ├──► registration-svc.nbis.internal (Registration)       │
│    └──► admin-svc.nbis.internal        (Admin Service)      │
│                                                              │
│  Auth Service                                                │
│    └──► (No sync calls to other services — uses DynamoDB    │
│          and S3 directly via AWS SDK)                        │
│                                                              │
│  eKYC Service                                                │
│    └──► auth-svc.nbis.internal/v1/auth/biometric            │
│         (calls Auth Service to verify before returning KYC) │
│                                                              │
│  Credential Service                                          │
│    └──► (No sync service calls — triggered by SQS)          │
│                                                              │
│  Admin Service                                               │
│    └──► auth-svc.nbis.internal/v1/admin/invalidate-session  │
│         (revoke active sessions on identity suspension)      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Service URL configuration

```
┌──────────────────────────────────────────────────────────────┐
│         Service URL Reference — NBIS                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  All URLs are environment variables — never hardcoded:      │
│                                                              │
│  # application.yml (Spring Boot)                            │
│  services:                                                   │
│    auth:       ${AUTH_SERVICE_URL}                          │
│    ekyc:       ${EKYC_SERVICE_URL}                          │
│    biometric:  ${BIOMETRIC_SERVICE_URL}                     │
│    resident:   ${RESIDENT_SERVICE_URL}                      │
│    credential: ${CREDENTIAL_SERVICE_URL}                    │
│    admin:      ${ADMIN_SERVICE_URL}                         │
│                                                              │
│  # ECS Task Definition environment variables                │
│  AUTH_SERVICE_URL      = http://auth-svc.nbis.internal      │
│  BIOMETRIC_SERVICE_URL = http://biometric-svc.nbis.internal │
│  EKYC_SERVICE_URL      = http://ekyc-svc.nbis.internal      │
│                                                              │
│  Why env vars?                                               │
│  ├── Dev: http://localhost:8081                             │
│  ├── Staging: http://auth-svc.nbis-staging.internal        │
│  └── Production: http://auth-svc.nbis.internal             │
│  Same code, different environments, different URLs          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 13.9 Load Balancing Strategies

The internal ALB supports multiple load balancing algorithms. NBIS uses round-robin for most services:

```
┌──────────────────────────────────────────────────────────────┐
│          Load Balancing Algorithms                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Round-robin (default):                                      │
│  Request 1 → Task A                                          │
│  Request 2 → Task B                                          │
│  Request 3 → Task C                                          │
│  Request 4 → Task A (back to start)                         │
│                                                              │
│  Use for: Auth Service, eKYC, Resident Services             │
│  (stateless — any task can handle any request)              │
│                                                              │
│  Least outstanding requests:                                 │
│  Route to whichever task has fewest in-flight requests      │
│                                                              │
│  Use for: Biometric Service (some requests take longer)     │
│  (avoids overloading one task with slow biometric checks)   │
│                                                              │
│  Sticky sessions (least used in NBIS):                       │
│  Same client always routed to same task                     │
│  Use only if service has local state (avoid this design)    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 13.10 Graceful Shutdown and Deregistration

When a service is being updated or scaled down, it must finish current requests before being removed from the load balancer. This is **graceful shutdown**.

```
┌──────────────────────────────────────────────────────────────┐
│              Graceful Shutdown Flow                          │
└──────────────────────────────────────────────────────────────┘

ECS decides to stop Task A (new deployment / scale down):

Step 1: ECS sends SIGTERM to container
Step 2: Spring Boot begins graceful shutdown:
        ├── Stops accepting new requests
        ├── ALB drains: waits for in-flight requests to finish
        │   (deregistrationDelay: 30 seconds)
        └── Completes all active requests

Step 3: ALB stops routing to Task A
Step 4: Task A finishes remaining requests
Step 5: Container exits cleanly
Step 6: ECS deregisters from target group

Result:
✅ Zero dropped requests
✅ No 502 errors during deployment
✅ Citizens experience no interruption

Spring Boot graceful shutdown config:
  server:
    shutdown: graceful
  spring:
    lifecycle:
      timeout-per-shutdown-phase: 30s

ALB deregistration delay:
  TargetGroup → Attributes → Deregistration delay: 30 seconds
  (must match or exceed Spring Boot shutdown timeout)
```

---

## 13.11 Service Mesh (Advanced)

For very large NBIS deployments, a **service mesh** (AWS App Mesh) provides additional capabilities on top of basic service discovery:

```
┌──────────────────────────────────────────────────────────────┐
│              Service Mesh Capabilities                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Basic (Internal ALB — NBIS default):                        │
│  ├── Load balancing ✅                                       │
│  ├── Health checking ✅                                      │
│  └── Service DNS ✅                                         │
│                                                              │
│  Service Mesh (AWS App Mesh — for large deployments):        │
│  ├── All of the above, plus:                                │
│  ├── mTLS between all services (automatic)                  │
│  ├── Circuit breaker (built into mesh, not service code)    │
│  ├── Retry logic (configured at mesh level)                 │
│  ├── Traffic shaping (send 10% to new version, 90% to old) │
│  ├── Distributed tracing (X-Ray integration)               │
│  └── Observability (metrics for every service pair)        │
│                                                              │
│  When to consider App Mesh for NBIS:                        │
│  ├── > 20 microservices                                     │
│  ├── mTLS between ALL services required by compliance       │
│  ├── Complex canary deployment requirements                 │
│  └── Need traffic metrics per service-to-service pair       │
│                                                              │
│  For < 10 services: Internal ALB + Cloud Map is sufficient  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 13.12 Failure Scenarios and Recovery

Service discovery must handle failures gracefully. Here are the key failure modes and how NBIS handles them:

```
┌──────────────────────────────────────────────────────────────┐
│          Failure Scenarios and Recovery                      │
├──────────────────────────┬───────────────────────────────────┤
│ Failure                  │ What happens                     │
├──────────────────────────┼───────────────────────────────────┤
│ Single ECS task crashes  │ ALB health check detects failure │
│                          │ Task removed from rotation in    │
│                          │ ~30 seconds                      │
│                          │ ECS launches replacement task    │
│                          │ Service continues on remaining   │
│                          │ tasks (no citizen impact if ≥2)  │
├──────────────────────────┼───────────────────────────────────┤
│ ALL tasks for a service  │ ALB has no healthy targets       │
│ crash simultaneously     │ Requests return 503              │
│                          │ ECS launches new tasks           │
│                          │ Service recovers in 60–120s      │
│                          │ DLQ catches async failures       │
│                          │ Sync calls: circuit breaker      │
├──────────────────────────┼───────────────────────────────────┤
│ ALB itself fails         │ Multi-AZ ALB: cross-AZ failover  │
│                          │ automatic (AWS managed)          │
├──────────────────────────┼───────────────────────────────────┤
│ Service too slow         │ ALB health check timeout = fail  │
│ (health check times out) │ Task removed from rotation       │
│                          │ Other tasks absorb load          │
│                          │ Auto-scaling triggered           │
├──────────────────────────┼───────────────────────────────────┤
│ New deployment breaks    │ minimumHealthyPercent: 100       │
│ health check             │ New tasks must pass health check │
│                          │ before old ones are stopped      │
│                          │ Failed deploy → ECS rolls back   │
└──────────────────────────┴───────────────────────────────────┘
```

---

## 13.13 Circuit Breaker Integration

Service discovery tells you where a service is. A **circuit breaker** tells you when to stop calling it temporarily — protecting against cascade failures.

```
┌──────────────────────────────────────────────────────────────┐
│         Circuit Breaker with Service Discovery               │
└──────────────────────────────────────────────────────────────┘

Normal state (CLOSED):
  Auth Service → biometric-svc.nbis.internal → success
  Auth Service → biometric-svc.nbis.internal → success
  Auth Service → biometric-svc.nbis.internal → success

Biometric Service becomes slow / erroring:
  Auth Service → biometric-svc.nbis.internal → timeout
  Auth Service → biometric-svc.nbis.internal → timeout
  Auth Service → biometric-svc.nbis.internal → timeout (5 failures)

Circuit OPENS (Resilience4j / Spring Cloud Circuit Breaker):
  Auth Service → circuit breaker → OPEN → fail fast
  (does not even attempt to call Biometric Service)
  Returns: { error: "Biometric service temporarily unavailable" }

After 30 seconds → circuit enters HALF-OPEN:
  Auth Service → tries ONE call to biometric-svc.nbis.internal
  ├── SUCCESS → circuit CLOSES (back to normal)
  └── FAILURE → circuit stays OPEN (another 30s)

Why this matters:
  Without circuit breaker:
  Auth Service waits 30s for each timeout
  → Threads pile up → Auth Service itself runs out of threads
  → Cascade failure: Auth goes down too

  With circuit breaker:
  Auth Service fails fast (< 1ms)
  → Keeps Auth Service healthy
  → Biometric Service recovers without dragging Auth down
```

---

## 13.14 Complete NBIS Service Discovery Architecture

```
┌──────────────────────────────────────────────────────────────┐
│          NBIS Service Discovery — Full Architecture          │
└──────────────────────────────────────────────────────────────┘

                    EXTERNAL INTERNET
                          │
                          │ HTTPS
                          ▼
                    API GATEWAY
                    (public — managed service)
                          │
                          │ VPC Link
                          ▼
              ┌───────────────────────┐
              │  INTERNAL ALBs        │  ← private subnet
              │  (one per service)    │     no public IP
              └───────────────────────┘
                    │         │
           ┌────────┘         └─────────┐
           ▼                            ▼
  auth-svc.nbis.internal    ekyc-svc.nbis.internal
  ┌──────────────────┐      ┌──────────────────────┐
  │ Auth Service ALB │      │ eKYC Service ALB     │
  └──────┬─────┬─────┘      └──────┬──────┬────────┘
         │     │                   │      │
       Task1 Task2               Task1  Task2
       AZ-a  AZ-b                AZ-a   AZ-b

         │ (AWS SDK — not service discovery)
         ▼
  ┌─────────────────────────────────────────────┐
  │  AWS Managed Services (no discovery needed) │
  │  DynamoDB, S3, KMS, SQS, ElastiCache        │
  │  (accessed via AWS SDK + IAM role)           │
  └─────────────────────────────────────────────┘

Cloud Map (lightweight internal DNS):
  nbis.local namespace
  ├── admin-svc.nbis.local   → Admin Service tasks
  ├── notif-svc.nbis.local   → Notification Service tasks
  └── partner-svc.nbis.local → Partner Management tasks
```

---

## 13.15 AWS Services Used

```
┌──────────────────────────────────────────────────────────────┐
│          AWS Services for Service Discovery in NBIS          │
├──────────────────────────────┬───────────────────────────────┤
│ AWS Service                  │ Role                         │
├──────────────────────────────┼───────────────────────────────┤
│ Internal ALB                 │ Primary service endpoint     │
│                              │ Load balancing + health check│
├──────────────────────────────┼───────────────────────────────┤
│ ECS Service                  │ Manages task lifecycle       │
│                              │ Auto-registers with ALB TG   │
├──────────────────────────────┼───────────────────────────────┤
│ ALB Target Group             │ Pool of healthy tasks        │
│                              │ Health check configuration   │
├──────────────────────────────┼───────────────────────────────┤
│ Route 53 (private zone)      │ DNS resolution for           │
│                              │ *.nbis.internal names        │
├──────────────────────────────┼───────────────────────────────┤
│ AWS Cloud Map                │ DNS-based service registry   │
│                              │ For lightweight internal svc │
├──────────────────────────────┼───────────────────────────────┤
│ AWS App Mesh (optional)      │ Service mesh for large       │
│                              │ deployments (mTLS, tracing)  │
├──────────────────────────────┼───────────────────────────────┤
│ CloudWatch                   │ ALB target health metrics    │
│                              │ UnHealthyHostCount alarm     │
└──────────────────────────────┴───────────────────────────────┘
```

---

## 13.16 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Service Discovery Reference               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP uses Kubernetes for container orchestration          │
│  (self-hosted deployments)                                  │
│                                                              │
│  Kubernetes service discovery:                              │
│  ├── Services defined as Kubernetes Service objects         │
│  ├── Stable DNS: service-name.namespace.svc.cluster.local  │
│  ├── kube-proxy handles load balancing to pods             │
│  └── Readiness probes = our ALB health checks              │
│                                                              │
│  MOSIP Kubernetes → NBIS AWS mapping:                        │
│  ┌──────────────────────────────┬───────────────────────┐   │
│  │ Kubernetes concept           │ AWS NBIS equivalent   │   │
│  ├──────────────────────────────┼───────────────────────┤   │
│  │ Service (ClusterIP)          │ Internal ALB          │   │
│  │ Pod (container instance)     │ ECS Task              │   │
│  │ Deployment (replica set)     │ ECS Service           │   │
│  │ Readiness probe              │ ALB health check      │   │
│  │ Liveness probe               │ ECS container check   │   │
│  │ Namespace DNS                │ Route 53 private zone │   │
│  │ Horizontal Pod Autoscaler    │ ECS auto-scaling      │   │
│  └──────────────────────────────┴───────────────────────┘   │
│                                                              │
│  In AWS deployments of MOSIP:                               │
│  Kubernetes → replaced with ECS + Internal ALB + Cloud Map  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 13.17 Key Terms

| Term | Definition |
|------|-----------|
| **Service discovery** | Mechanism for a service to find the network location of another service |
| **Service registry** | Database of all running service instances and their health status |
| **Internal ALB** | Application Load Balancer inside a VPC — not internet-accessible |
| **Target group** | Pool of ECS tasks registered with an ALB for load balancing |
| **Health check** | Periodic probe to determine if a service instance is healthy |
| **Liveness check** | Is the process alive? If not → restart the container |
| **Readiness check** | Is the service ready for traffic? If not → remove from ALB |
| **Deregistration delay** | Time ALB waits for in-flight requests to complete before removing a task |
| **Graceful shutdown** | Finishing current requests cleanly before stopping a service |
| **AWS Cloud Map** | AWS service registry with DNS-based service discovery |
| **Route 53 private zone** | DNS zone visible only inside the VPC (e.g. nbis.internal) |
| **Circuit breaker** | Stops calling a failing service temporarily to prevent cascade failure |
| **Resilience4j** | Java library implementing circuit breaker pattern for Spring Boot |
| **AWS App Mesh** | Service mesh providing mTLS, tracing, and traffic shaping |
| **VPC Link** | API Gateway connection to internal ALB inside the VPC |
| **Round-robin** | Load balancing algorithm distributing requests evenly across targets |
| **Least outstanding requests** | ALB algorithm routing to the target with fewest active connections |
| **SIGTERM** | Linux signal sent to a process to request graceful shutdown |
| **minimumHealthyPercent** | ECS deployment setting ensuring minimum tasks stay healthy during update |

---

## 13.18 Key Takeaways

- **Service discovery solves the dynamic IP problem** — in containerized environments, IP addresses change constantly. Never hardcode them. Use stable DNS names backed by ALB or Cloud Map.
- **Internal ALB is the primary NBIS pattern** — it provides server-side load balancing, health checking, and a stable DNS name. ECS auto-registers and deregisters tasks without any manual steps.
- **Health checks are the foundation** — service discovery is only useful if it routes to healthy instances. Use liveness checks for restart decisions and readiness checks for ALB routing decisions.
- **Graceful shutdown prevents dropped requests** — always configure Spring Boot graceful shutdown and match the ALB deregistration delay. This ensures zero-downtime deployments.
- **Circuit breakers complement service discovery** — discovery tells you where a service is, but does not tell you when to stop calling it. Circuit breakers protect against cascade failures when a service becomes slow or unavailable.
- **Use environment variables for service URLs** — same Docker image deployed to dev, staging, and production with different URLs injected at runtime. Never hardcode service addresses.
- **Kubernetes service discovery = ECS + Internal ALB** — if you understand MOSIP's Kubernetes service discovery model, the AWS ALB model maps directly to it concept for concept.
- **AWS-managed services need no discovery** — DynamoDB, S3, SQS, and KMS are accessed via the AWS SDK using IAM roles. There are no IP addresses or hostnames to manage.

---

## 13.19 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 14 | Caching — ElastiCache Redis for OTP, sessions, and hot-path optimization |
| Chapter 15 | High Availability — Multi-AZ, auto-scaling, failover strategies |
| Chapter 16 | Disaster Recovery — RTO, RPO, backup strategies, cross-region planning |

---

*Chapter 13 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
