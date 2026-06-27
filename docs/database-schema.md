# Database Schema & ERD Specifications

This project implements a **Database-Per-Service** microservice architecture pattern. To maintain loose coupling, the **Auth Service** and **Queue Service** use completely separate PostgreSQL databases (hosted on Serverless Neon Postgres). They share no tables and communicate solely through REST APIs and AMQP message brokers (RabbitMQ).

---

## 1. Auth Service Database Schema

The Auth Database handles identity, session storage, and security audit logging.

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

enum RoleType {
  PATIENT
  DOCTOR
  RECEPTIONIST
  ADMIN
}

model Role {
  id        String   @id @default(cuid())
  name      RoleType @unique
  users     User[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model User {
  id                 String              @id @default(cuid())
  fullName           String
  email              String              @unique
  password           String?
  isVerified         Boolean             @default(false)
  roleId             String
  role               Role                @relation(fields: [roleId], references: [id])
  sessions           Session[]
  refreshTokens      RefreshToken[]
  auditLogs          AuditLog[]
  createdAt          DateTime            @default(now())
  updatedAt          DateTime            @updatedAt
  verificationToken  VerificationToken?
  passwordResetToken PasswordResetToken?

  @@index([email])
}

model Session {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([userId])
}

model RefreshToken {
  id        String   @id @default(cuid())
  token     String   @unique
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  expiresAt DateTime
  createdAt DateTime @default(now())

  @@index([userId])
}

model AuditLog {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  action    String
  createdAt DateTime @default(now())

  @@index([userId])
}

model VerificationToken {
  id        String   @id @default(cuid())
  token     String   @unique
  userId    String   @unique
  user      User     @relation(fields: [userId], references: [id])
  expiresAt DateTime
  createdAt DateTime @default(now())

  @@index([token])
}

model PasswordResetToken {
  id        String   @id @default(cuid())
  token     String   @unique
  userId    String   @unique
  user      User     @relation(fields: [userId], references: [id])
  expiresAt DateTime
  createdAt DateTime @default(now())

  @@index([token])
}
```

---

## 2. Queue Service Database Schema

The Queue Database manages organizational units (hospitals, departments), professional profiles (doctors), active queues, and historical/real-time queue entries.

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

enum QueueStatus {
  WAITING
  IN_PROGRESS
  COMPLETED
  SKIPPED
  CANCELLED
}

model Hospital {
  id          String       @id @default(cuid())
  name        String
  address     String
  city        String
  departments Department[]
  doctors     Doctor[]
  queues      Queue[]
  createdAt   DateTime     @default(now())
  updatedAt   DateTime     @updatedAt
}

model Department {
  id         String   @id @default(cuid())
  name       String
  hospitalId String
  hospital   Hospital @relation(fields: [hospitalId], references: [id])
  doctors    Doctor[]
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  @@index([hospitalId])
}

model Doctor {
  id                      String     @id @default(cuid())
  fullName                String
  email                   String?    @unique
  userId                  String?    @unique // References Auth User CUID asynchronously
  hospitalId              String
  departmentId            String
  averageConsultationTime Int        @default(5)
  department              Department @relation(fields: [departmentId], references: [id])
  hospital                Hospital   @relation(fields: [hospitalId], references: [id])
  queue                   Queue?
  createdAt               DateTime   @default(now())
  updatedAt               DateTime   @updatedAt

  @@index([hospitalId])
  @@index([departmentId])
}

model Queue {
  id         String       @id @default(cuid())
  name       String
  hospitalId String
  doctorId   String       @unique
  isOpen     Boolean      @default(true)
  doctor     Doctor       @relation(fields: [doctorId], references: [id])
  hospital   Hospital     @relation(fields: [hospitalId], references: [id])
  entries    QueueEntry[]
  createdAt  DateTime     @default(now())
  updatedAt  DateTime     @updatedAt

  @@index([hospitalId])
}

model QueueEntry {
  id          String      @id @default(cuid())
  patientId   String                  // References Auth User CUID asynchronously
  queueId     String
  tokenNumber Int
  status      QueueStatus @default(WAITING)
  joinedAt    DateTime    @default(now())
  queue       Queue       @relation(fields: [queueId], references: [id])
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt

  @@unique([queueId, tokenNumber])
  @@index([queueId])
  @@index([patientId])
}
```

---

## 3. Database Design Choices & Performance Optimization

### A. Loose Coupling with Reference Keys
- Instead of direct foreign keys connecting `QueueEntry.patientId` to `User.id` (which is impossible across database boundaries), the Queue Service stores the user's `CUID` string as a reference key (`patientId`).
- This allows services to run on separate physical hosts, database engines, or database clusters.

### B. Index Optimizations
- **Auth DB**: Indexes are created on `User.email` to speed up registration checks and credential logins, and on token tables (`VerificationToken.token`, `PasswordResetToken.token`) to optimize link verification routing.
- **Queue DB**: Unique composite constraints on `QueueEntry(queueId, tokenNumber)` guarantee that no two patients can be assigned the exact same token number in a single queue. Indexes on foreign keys (`hospitalId`, `departmentId`, `queueId`) speed up read performance under heavy query loads.
