# API Specification

All routes are versioned and separated by the specific microservice domain.

---

## 🔐 1. Auth Service API

**Base URL (Prod):** `https://hospital-auth-service-yynk.onrender.com/api/v1/auth`

### POST `/register`
Registers a new Patient user account.

**Request Body:**
```json
{
  "fullName": "Jane Doe",
  "email": "jane@example.com",
  "password": "SecurePassword123!"
}
```
**Response (201 Created):**
```json
{
  "status": "success",
  "message": "User registered. Please check email for verification link.",
  "data": {
    "userId": "cldh1234abcd5678"
  }
}
```

### POST `/login`
Authenticates user. Returns access token in response body, and sets a Secure HttpOnly Cookie with the refresh token.

**Request Body:**
```json
{
  "email": "jane@example.com",
  "password": "SecurePassword123!"
}
```
**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsIn...",
    "user": {
      "id": "cldh1234abcd5678",
      "fullName": "Jane Doe",
      "email": "jane@example.com",
      "role": "PATIENT"
    }
  }
}
```
**Headers Set:**
- `Set-Cookie: refreshToken=eyJhb...; Path=/api/v1/auth; HttpOnly; Secure; SameSite=None; Max-Age=604800`

### POST `/refresh`
Rotates refresh tokens and issues a new access token. Requires valid HttpOnly cookie and CSRF validation token.

**Headers Required:**
- `x-csrf-token`: string

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsIn..."
  }
}
```

### POST `/logout`
Invalidates the current session and revokes the refresh token from the database.

**Response (200 OK):**
```json
{
  "status": "success",
  "message": "Logged out successfully"
}
```

---

## 🏥 2. Queue Service API

**Base URL (Prod):** `https://hospital-queue-service.onrender.com/api/v1`

### POST `/queue-entries/join`
Join a hospital queue. Restricted to users with the `PATIENT` role.

**Headers Required:**
- `Authorization: Bearer <accessToken>`

**Request Body:**
```json
{
  "queueId": "cldh8888queueId"
}
```
**Response (201 Created):**
```json
{
  "status": "success",
  "data": {
    "id": "cldh9999entryId",
    "patientId": "cldh1234abcd5678",
    "queueId": "cldh8888queueId",
    "tokenNumber": 14,
    "status": "WAITING",
    "joinedAt": "2026-06-27T12:00:00.000Z"
  }
}
```

### GET `/queue-entries/position/:id`
Retrieves live position details and estimated wait time for a specific queue entry. Publicly accessible.

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "queueEntryId": "cldh9999entryId",
    "tokenNumber": 14,
    "position": 3,
    "estimatedWaitMinutes": 15,
    "status": "WAITING"
  }
}
```

### PATCH `/queue-entries/call-next/:queueId`
Calls the next waiting patient in the queue. Restricted to `DOCTOR` or `RECEPTIONIST` assigned to the queue.

**Headers Required:**
- `Authorization: Bearer <accessToken>`

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "id": "cldh9999entryId",
    "tokenNumber": 14,
    "status": "IN_PROGRESS"
  }
}
```

### PATCH `/queue-entries/complete/:queueId`
Completes the current consultation. Restricted to `DOCTOR`.

**Headers Required:**
- `Authorization: Bearer <accessToken>`

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "id": "cldh9999entryId",
    "status": "COMPLETED"
  }
}
```

### GET `/queues/:id/dashboard`
Gets aggregated real-time dashboard data for a hospital queue.

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "queueId": "cldh8888queueId",
    "totalWaiting": 5,
    "inProgressToken": 13,
    "averageWaitTime": 5,
    "lastCalledToken": 13
  }
}
```
