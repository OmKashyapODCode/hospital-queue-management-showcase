# Hospital Queue Management System - Architecture Overview

This document provides a comprehensive technical overview of the system's architecture, microservices, communication patterns, databases, security protocols, and integration points.

---

## 1. High-Level System Architecture

The application is structured as a decentralized, event-driven microservices architecture. It decouples the core domain responsibilities (Identity/Access vs. Queue Operations) into independent deployable units.

```mermaid
graph TB
    %% Client Tier
    subgraph Client Tier [Frontend Application]
        client[React + Redux SPA]
    end

    %% Routing / Gateway
    subgraph Hosting / Deployment [Public Gateway]
        client -- REST (HTTPS) --> authService[Auth Service]
        client -- REST & WebSockets (WSS) --> queueService[Queue Service]
    end

    %% Services
    subgraph Microservices [Backend Microservices Node.js 24 / Express 5]
        authService
        queueService
    end

    %% Verification Client
    queueService -- "JWT Verification HTTP POST /me" --> authService

    %% Shared Databases & Caching
    subgraph Data & Cache Tier
        authDB[(Auth DB: PostgreSQL Neon)]
        queueDB[(Queue DB: PostgreSQL Neon)]
        redis[(Cache & Session Store: Redis Upstash)]
    end

    authService -- Prisma ORM --> authDB
    authService -- Cache Reads/Writes --> redis
    queueService -- Prisma ORM --> queueDB

    %% Async Broker
    subgraph Event Broker [Message Queue]
        rabbitmq{RabbitMQ CloudAMQP}
    end

    authService -- "Publish (auth.email queue)" --> rabbitmq
    queueService -- "Publish (domain events)" --> rabbitmq
    rabbitmq -- "Consume tasks" --> authService

    %% External Systems
    subgraph External Systems
        brevo[Brevo SMTP Email Service]
    end

    authService -- "Brevo API HTTPS POST" --> brevo

    %% Styling / Color Schemes
    classDef clientStyle fill:#3b82f6,stroke:#1d4ed8,stroke-width:2px,color:#fff;
    classDef serviceStyle fill:#8b5cf6,stroke:#6d28d9,stroke-width:2px,color:#fff;
    classDef dbStyle fill:#10b981,stroke:#047857,stroke-width:2px,color:#fff;
    classDef brokerStyle fill:#f59e0b,stroke:#d97706,stroke-width:2px,color:#fff;
    classDef extStyle fill:#ec4899,stroke:#be185d,stroke-width:2px,color:#fff;

    class client clientStyle;
    class authService,queueService serviceStyle;
    class authDB,queueDB,redis dbStyle;
    class rabbitmq brokerStyle;
    class brevo extStyle;
```

---

## 2. Visual Architecture Concept

Below is a generated visual concept mockup of the microservices system topology, showing the connection between the React Client, backend APIs, shared message queues, caching layers, and database clusters.

![Architectural topology mockup showing microservices connections, caching nodes, RabbitMQ broker, and databases.](C:/Users/omkas/.gemini/antigravity/brain/3213ca11-dcb5-4a53-bd8f-e4626b7a78f4/architecture_diagram_1782562143650.png)

---

## 3. Microservice Specifications

### [Auth Service](file:///C:/Users/omkas/Desktop/placement/hospital%20Queue%20Management/Backend/auth-service)
- **Role**: Identity Provider & Session Manager.
- **Port**: `5001` (Dev) / `https://hospital-auth-service-yynk.onrender.com` (Prod)
- **Database**: Dedicated PostgreSQL database schema for User Credentials, Verification Tokens, Password Reset Tokens, and Audit Logs.
- **Cache**: Redis for tracking CSRF secret tokens and rate limiting metrics.
- **Email Handler**: Contains a RabbitMQ consumer task runner that pulls events from the `auth.email` queue and dispatches verification/reset emails using the Brevo HTTP API.

### [Queue Service](file:///C:/Users/omkas/Desktop/placement/hospital%20Queue%20Management/Backend/queue-service)
- **Role**: Hospital Queue and Patient Flow Management.
- **Port**: `5002` (Dev) / `https://hospital-queue-service.onrender.com` (Prod)
- **Database**: Dedicated PostgreSQL database schema for Hospitals, Departments, Doctors, Queues, and Queue Entries.
- **WebSockets**: Hosts the Socket.io server to push live queue updates directly to patient dashboards.
- **Security Check**: For all protected API endpoints, it behaves as a Resource Server that verifies JWT tokens via internal REST calls back to Auth Service's `/me` endpoint.

---

## 4. Key Architectural Flows

### A. Real-time Patient Queue Updates

The client connects to Socket.io hosted on the Queue Service, joining a specific room matching the queue identifier. Actions taken by doctors or receptionists broadcast events to all users in the room.

```mermaid
sequenceDiagram
    autonumber
    actor Doctor
    participant Client as Frontend (Doctor View)
    participant QS as Queue Service (Express)
    participant DB as Queue Database (PostgreSQL)
    participant WS as Socket.io Server
    actor Patient
    participant PC as Frontend (Patient View)

    Patient->>WS: joinQueue(queueId)
    WS-->>Patient: Added to Room: queue:<queueId>
    
    Doctor->>Client: Click "Call Next Patient"
    Client->>QS: PATCH /api/v1/queue-entries/call-next/:queueId (Bearer JWT)
    Note over QS: Verifies JWT via Auth Service
    QS->>DB: Update next waiting entry to 'IN_PROGRESS'
    DB-->>QS: Return updated entry details
    QS->>WS: emitQueueEvent(queueId, "PATIENT_CALLED", updatedPatient)
    WS->>PC: Broadcast "PATIENT_CALLED" (to all clients in room)
    PC-->>Patient: React State Updates (Live Position/Estimation Changes)
    QS-->>Client: 200 OK (with updated patient details)
```

---

### B. Asynchronous Email Dispatch (RabbitMQ)

To avoid blocking HTTP responses on slow external SMTP or mail API operations, the Auth Service utilizes an event-driven publisher-consumer model.

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant AuthAPI as Auth Service (API Router)
    participant DB as Database (PostgreSQL)
    participant Broker as RabbitMQ (CloudAMQP)
    participant Consumer as Auth Service (Email Consumer)
    participant Brevo as Brevo HTTP API

    User->>AuthAPI: POST /api/v1/auth/register (Credentials)
    AuthAPI->>DB: Save User (status: PENDING_VERIFICATION)
    AuthAPI->>DB: Create Verification Token
    AuthAPI->>Broker: Publish Event {event: "SEND_VERIFICATION_EMAIL", email, verificationToken} to 'auth.email'
    Note over AuthAPI: Durable & Persistent Queue
    AuthAPI-->>User: 201 Created (HTTP Response returned immediately)
    
    Note over Broker: Queue contains message
    Broker->>Consumer: Deliver message (Prefetch 1)
    Consumer->>Brevo: POST /v3/smtp/email (native fetch API)
    alt Success (200/201 OK)
        Brevo-->>Consumer: Email Sent Confirmation
        Consumer->>Broker: ack (Acknowledge)
        Note over Broker: Message removed from queue
    else API Error / Timeout
        Brevo-->>Consumer: Failed (e.g. 503 Service Unavailable)
        alt Retry count < 3
            Consumer->>Broker: nack(requeue: false) + sendToQueue(auth.email, retryCount++)
            Note over Broker: Requeued with delay
        else Max retries exhausted (>= 3)
            Consumer->>Broker: nack(requeue: false)
            Note over Broker: Sent to Dead-Letter Queue (DLQ)
        end
    end
```

---

### C. Authentication and Token Rotation Lifecycle

Security is reinforced via **Refresh Token Rotation (RTR)**. Every time an access token is refreshed, the old refresh token is deleted, and a new key pair is generated.

```mermaid
sequenceDiagram
    autonumber
    participant Client as React Client (LocalStorage/HttpOnly Cookies)
    participant Auth as Auth Service
    participant DB as Auth Database (PostgreSQL)

    Client->>Auth: POST /api/v1/auth/refresh (Cookie: refreshToken, Header: x-csrf-token)
    Auth->>Auth: Verify JWT signature of Refresh Token
    Auth->>DB: Check if Refresh Token exists in database
    alt Token Exists (Valid Flow)
        Auth->>DB: DELETE old Refresh Token (Revoke immediately)
        Auth->>Auth: Generate new Access Token (15 mins)
        Auth->>Auth: Generate new Refresh Token (7 days)
        Auth->>DB: Save new Refresh Token to DB
        Auth-->>Client: Set-Cookie: refreshToken (HttpOnly, Secure) + Response Body: accessToken
    else Token Not Found (Replay/Attack Flow)
        Note over Auth: Token has already been used or deleted!
        Auth->>DB: DELETE all active refresh tokens for this User ID (Revocation of session)
        Auth-->>Client: 401 Unauthorized (Session Suspended)
    end
```

---

## 5. Security & Isolation Matrix

| Component | Security Feature | Implementation Details |
|---|---|---|
| **CORS Policy** | Whitelisted Origins | Restricted to client URL `hospital-queue-management-eosin.vercel.app` and `localhost:5173`. |
| **Storage Security** | HttpOnly Cookies | Refresh tokens are stored strictly inside HttpOnly cookies with `Secure`, `SameSite=None/Lax` flags, avoiding Javascript access (XSS mitigation). |
| **CSRF Defense** | double-submit token | State-changing endpoints (`/refresh`, `/logout`) require an `x-csrf-token` header, verified against a Redis-cached secret. |
| **Microservice Auth** | Inter-service JWT | Queue Service makes a backend request `/me` to the Auth service passing the Bearer token to authorize patient/doctor credentials. |
| **API Protection** | Express Rate Limiting | Implemented on registration and login endpoints to safeguard against credential stuffing. |
| **HTTP Security Headers** | Helmet.js | Applies HTTP headers to avoid standard clickjacking, MIME sniffing, and scripting vulnerabilities. |

---

## 6. Infrastructure & Deployment Setup

- **Frontend Hosting**: Deployed to Vercel with automatic builds triggered on pushing to the `main` branch.
- **Backend API Hosting**: Auth Service and Queue Service are hosted as web services on Render.
- **Relational Storage**: Dual PostgreSQL instances hosted on serverless Neon Postgres.
- **Caching & Brokerage**:
  - Redis instances run on Upstash.
  - RabbitMQ AMQP message broker hosted on CloudAMQP.
