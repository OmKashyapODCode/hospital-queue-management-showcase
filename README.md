# Hospital Queue Management System (Microservices)

A full-stack, production-grade hospital queue management platform built with a microservices architecture. The system enables patients to join digital queues remotely, doctors to call and process patient flows in real-time, and administrators to manage hospitals and analyze operational bottlenecks.

**Live Demo:** [hospital-queue-management-eosin.vercel.app](https://hospital-queue-management-eosin.vercel.app)

---

## 🛠️ Tech Stack & Microservices

The application consists of two decoupled Node.js services communicating asynchronously via a message broker and serving a single-page React client.

### System Architecture
```
┌─────────────────────────────────────────────┐
│               Frontend Client               │
│          React + Redux + Socket.io          │
│          (Deployed on Vercel)               │
└────────────┬──────────────────┬────────────┘
             │ REST (HTTPS)     │ WebSockets (WSS)
┌────────────▼──────┐  ┌───────▼────────────┐
│   Auth Service    │  │   Queue Service    │
│   Node / Express  │  │   Node / Express   │
│   Port 5001       │  │   Port 5002        │
│   Render Hosting  │  │   Render Hosting   │
└──────┬────────────┘  └──────┬─────────────┘
       │                      │
┌──────▼──────────────────────▼─────────────┐
│            Shared Infrastructure          │
│  PostgreSQL (Neon)  ·  Redis (Upstash)    │
│  RabbitMQ (CloudAMQP Broker)              │
└───────────────────────────────────────────┘
```

- **Frontend**: React 18, Vite, Redux Toolkit, Socket.io-client, Axios, Tailwind CSS.
- **Auth Microservice**: Node.js 24, Express 5, Prisma ORM, PostgreSQL, Redis, RabbitMQ.
- **Queue Microservice**: Node.js 24, Express 5, Socket.io 4, Prisma ORM, PostgreSQL, RabbitMQ.
- **Infrastructure**: Neon Serverless PostgreSQL (decoupled schemas), Upstash Redis (caching/sessions), CloudAMQP RabbitMQ.

---

## 🚀 Key Architectural Highlights

### ⚡ Real-Time Patient Updates (Socket.io)
When a doctor updates a queue (e.g. calls the next patient or completes a consultation), the Queue Service publishes a WebSocket event. The frontend connects to unique rooms scoped by `queueId`, updating patient dashboards and estimated wait times instantly without resource-heavy client polling.

### ✉️ Event-Driven Email Pipeline (RabbitMQ)
Registration and password reset flows bypass the request-response cycle. The Auth Service publishes a serialized email job to RabbitMQ and returns a `201/200` response to the user under `50ms`. A consumer processes the jobs asynchronously, delivering transactional emails via the Brevo API. It features prefetch controls, manual acknowledgments (`ack`), and automatic exponential retries (up to 3 attempts) before dead-lettering.

### 🔐 Production-Grade Authentication & Security
- **Refresh Token Rotation (RTR)**: Refresh tokens are single-use. Re-using a refresh token triggers automatic revocation of all active sessions for that user to prevent replay attacks.
- **HttpOnly Cookies**: Refresh tokens are stored in strict `HttpOnly`, `Secure`, `SameSite=None` cookies to mitigate Cross-Site Scripting (XSS) risks.
- **CSRF Mitigation**: Double-submit cookie pattern requires an `x-csrf-token` header on state-changing endpoints, verified against a Redis-based cache.

---

## 📂 Repository Contents

This showcase repository highlights the architectural specifications and database schemas without exposing proprietary application source code:

- 📄 [Database Schemas & ERD](./docs/database-schema.md) — Isolated PostgreSQL designs using Prisma.
- 📄 [API Specifications](./docs/api-specification.md) — RESTful API routing, parameters, and payloads.
- 📄 [System Design & Failure Modes](./docs/system-design.md) — Fault tolerance, microservice integration, and state consistency.

---

## 🔒 Security & CORS Matrix

| Layer | Protocol | Purpose |
|---|---|---|
| **CORS Policy** | Whitelisted Domain Check | Restricts HTTP/WebSocket origins to the production domain. |
| **JWT Access** | Temporary Bearer tokens (15m) | Handled in-memory on the client to avoid local storage theft. |
| **Audit Logging** | Winston Logger + DB Records | Captures every high-impact auth action for security audits. |
| **API Protection** | Express rate-limiter middleware | Defends public auth endpoints against brute-force attacks. |
