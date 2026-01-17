# Intelligent Notification System – Architecture

## Overview
This system provides a reliable, scalable way for client applications (e.g., Swiggy, Uber) to send notifications to end users via multiple channels (Push, Email, SMS). It is designed with **clear synchronous and asynchronous boundaries**, **failure-aware processing**, and **real-world delivery guarantees**.

The system does **not** guarantee exactly-once delivery. Instead, it provides at-least-once delivery for critical notifications and at-most-once delivery for non-critical notifications.

---

## High-Level Architecture Diagram (Logical)

```
┌──────────────┐
│ Client App   │
│ (Swiggy etc) │
└──────┬───────┘
       │  POST /notifications
       ▼
┌──────────────────────────┐
│ Notification API Service │
│ - Auth & validation      │
│ - Idempotency check      │
│ - Rule Engine            │
└──────┬───────────────────┘
       │ Persist state = ACCEPTED
       ▼
┌──────────────────────────┐
│ Notification DB          │
│ - Notification           │
│ - NotificationStatus     │
│ - UserDevice             │
└──────┬───────────────────┘
       │ Enqueue
       ▼
┌──────────────────────────┐
│ Message Queue            │
│ (Kafka / RabbitMQ)       │
└──────┬───────────────────┘
       │ Consume
       ▼
┌──────────────────────────┐
│ Delivery Workers         │
│ - Retry logic            │
│ - Failure classification│
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│ Channel Adapters         │
│ - Push (FCM/APNS)        │
│ - Email / SMS            │
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│ External Providers       │
│ (FCM / APNS / SMTP)      │
└──────────────────────────┘
```

---

## Component Responsibilities

### 1. Client Application
- Backend systems that want to notify users (not mobile apps)
- Sends notification requests with a unique `requestId`
- Can safely retry requests without causing duplicates

---

### 2. Notification API Service (Synchronous)
**Responsibilities:**
- Authenticate and authorize client
- Validate request schema
- Enforce idempotency using `requestId`
- Execute rule engine
- Persist initial notification state
- Enqueue notification for async processing
- Return `202 Accepted`

**Does NOT:**
- Send notifications
- Talk to external providers
- Perform retries

---

### 3. Rule Engine (Synchronous)
**Purpose:**
- Decide whether a notification should be sent
- Determine channels, priority, and retry policy

**Examples:**
- User opted out
- Quiet hours
- Drop non-critical notifications

---

### 4. Notification Database (Synchronous)
**Stores:**
- Notification (current state)
- NotificationStatus (state transitions)
- UserDevice (device tokens)
- ClientApplication metadata

**Why synchronous:**
- ACCEPTED implies durability
- DB write failure means request failure

---

### 5. Message Queue (Asynchronous Boundary)
**Purpose:**
- Decouple request handling from delivery
- Absorb traffic spikes
- Enable retries

**Key detail:**
- Enqueue is synchronous
- Consumption is asynchronous

---

### 6. Delivery Workers (Asynchronous)
**Responsibilities:**
- Consume messages from queue
- Move state to IN_PROGRESS
- Call channel adapters
- Retry retryable failures
- Decide final state: DELIVERED or FAILED

---

### 7. Channel Adapters (Synchronous within worker)
**Purpose:**
- Abstract provider-specific logic
- Provide uniform interface to workers

**Examples:**
- PushChannelAdapter (FCM/APNS)
- EmailChannelAdapter (SMTP/SES)

---

### 8. External Providers
- FCM / APNS for push notifications
- SMTP / SES for email
- SMS gateways

**Design assumption:**
- External providers are unreliable
- System must handle timeouts, retries, and duplicates

---

## Notification Lifecycle States

```
RECEIVED → ACCEPTED → ENQUEUED → IN_PROGRESS → DELIVERED | FAILED
```

- ACCEPTED means the system has taken responsibility
- DELIVERED and FAILED are both terminal states

---

## Key Design Principles
- Asynchronous processing for unreliable operations
- Idempotency at API boundary
- Explicit failure modeling
- Clear ownership of data
- Separation of concerns

---

## Assumptions
- Client applications are trusted backends (mobile apps do not call this system directly).
- Clients generate a unique `requestId` per logical notification to enable idempotency.
- External providers (FCM/APNS/SMTP) are unreliable and may be slow, return transient errors, or deliver duplicates.
- Token lifecycle is controlled by client apps and OS; this system reacts to token changes but does not control them.
- Exactly-once delivery is not guaranteed; delivery semantics vary by notification priority.

---

## Non-Goals
- Real-time delivery guarantees (no hard latency SLAs).
- Exactly-once delivery across external providers.
- Owning full user profiles or mobile app logic.
- Provider-side analytics or device reachability checks.
- End-user UI for managing preferences (assumed to be handled by client apps).

---

## Failure Scenarios & Handling

### 1. Client Retries the Same Request
- **Cause:** Network timeout before client receives response.
- **Handling:** Idempotency enforced via `requestId`; duplicate requests return existing state without re-enqueueing.

### 2. Database Write Fails During Acceptance
- **Cause:** DB outage or transient error.
- **Handling:** Request is rejected with `5xx`; no acceptance or enqueue occurs.

### 3. Queue Publish Fails After Acceptance
- **Cause:** Broker unavailable.
- **Handling:** Transaction/compensation ensures state is not advanced to ENQUEUED; request fails fast or is retried internally.

### 4. Worker Crash During Processing
- **Cause:** Node failure, OOM, deployment.
- **Handling:** Message is re-delivered by the queue; processing resumes (at-least-once semantics).

### 5. External Provider Transient Failure
- **Cause:** Network timeout, 5xx, rate limiting.
- **Handling:** Retry with backoff up to `maxRetries`; state remains `IN_PROGRESS`.

### 6. External Provider Permanent Failure (Invalid Token)
- **Cause:** App uninstalled, token expired, NotRegistered.
- **Handling:** Mark notification `FAILED`; mark `UserDevice` as `INVALID`; stop retries for that token.

### 7. Duplicate Delivery
- **Cause:** Retries or provider behavior.
- **Handling:** Accepted for critical notifications; minimized for non-critical via retry policy and idempotent request handling.

---

## Interview Summary (One-Liner)
> This system synchronously validates and accepts notification requests with idempotency and rules, persists durable state, and processes delivery asynchronously via queues and workers with explicit failure classification and retries.

