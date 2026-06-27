# System Design & Failure Mode Analysis

This document outlines key technical decisions made during the design of the Hospital Queue Management System, focusing on reliability, data consistency, security, and mitigation of common system failures.

---

## 1. Microservice Integration & Token Validation

To maintain independent databases while ensuring authenticated routes, the system runs a Resource Server verification model.

```
┌──────────────┐         1. API Request (Bearer Token)          ┌───────────────┐
│  React App   ├───────────────────────────────────────────────>│ Queue Service │
└──────────────┘                                                └───────┬───────┘
       ▲                                                                │
       │                                                                │ 2. POST /me
       │ 4. HTTP Response                                               │    (Validate Token)
       │                                                                ▼
┌──────┴───────┐                                                ┌───────────────┐
│ Patient View │<───────────────────────────────────────────────┤ Auth Service  │
└──────────────┘             3. Verified User Claims            └───────────────┘
```

1. **Authentication Gateway**: All operations involving modifying state (joining queues, calling patients) require verification.
2. **Dynamic Validation**: Instead of the Queue Service performing decoupled/local JWT decoding (which is fast but vulnerable to token revocation latency), the Queue Service calls the Auth Service’s `/me` endpoint asynchronously for each sensitive route.
3. **Caching Strategy**: Plans are in place to introduce a short-lived cache (e.g., Redis-based token TTL caching) in the Queue Service to reduce inter-service latency under peak traffic loads.

---

## 2. Event-Driven Asynchronous Processing (RabbitMQ)

To avoid blocking web operations (like user registration) with third-party service calls (sending email via SMTP/Brevo APIs), task delegating is decoupled.

- **Publisher Pattern**: When `POST /register` is hit, the database record is inserted, the verification token is saved, and a message containing the payload is published to the `auth.email` queue. The client immediately receives a `201 Created` code.
- **Consumer Control**: The email consumer uses `prefetch(1)` to specify how many messages are fetched before being acknowledged. This provides backpressure safety, preventing Node.js from being overwhelmed by API rate limits from Brevo.
- **Reliable Retry Mechanism**: If the Brevo API returns a transient error (e.g. `503 Service Unavailable` or connection timeout), the consumer extracts the headers, increments `x-retry-count`, sends a negative acknowledgment (`nack`) without requeuing, and republishes it with persistence. This prevents immediate broker loops.

---

## 3. Failure Mode & Resilience Planning

Here is how the system handles critical infrastructure degradation:

### A. RabbitMQ Broker Outage
- **Symptom**: CloudAMQP becomes unreachable.
- **Fallback**: The services are designed to boot up even if RabbitMQ connections fail (non-fatal start). If a request requires publishing a job, the system catches the exception and logs it via Winston. User registration succeeds immediately, but the email delivery is logged as "Degraded/Pending" and retried once the message broker is back online.

### B. Redis Cluster Downtime
- **Symptom**: Upstash Redis is down, causing session check issues and CSRF lookup failure.
- **Mitigation**: A memory-fallback layer can be toggled via environment variables. If Redis is unavailable, Auth fallback endpoints can dynamically default back to in-memory maps or standard validation arrays for short-lived requests, maintaining service uptime while losing horizontal scaling capabilities.

### C. Neon DB Serverless Cold Starts
- **Symptom**: Serverless database nodes scale down to zero after idle periods, causing the first query to take 3-5 seconds.
- **Mitigation**: Health endpoints `/health` ping the databases periodically to keep DB instances warm during active clinic operational hours.
