# MedReceipt — Digital Prescription Management System

A full-stack, clinical-grade prescription management platform built with **React**, **Node.js/Express**, and **MongoDB**. Designed with a layered security architecture including bcrypt password hashing, JWT session management, TOTP-based two-factor authentication, role-based access control, and an append-only audit log.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Environment Variables](#environment-variables)
- [Seed Accounts](#seed-accounts)
- [API Reference](#api-reference)
- [Security Architecture](#security-architecture)
- [Data Models](#data-models)
- [Role Capabilities](#role-capabilities)
- [Two-Factor Authentication](#two-factor-authentication)
- [Password Policy](#password-policy)
- [Audit Logging](#audit-logging)
- [Production Considerations](#production-considerations)

---

## Features

- **Three-role system** — Admin, Doctor, Patient — each with strictly scoped access
- **TOTP two-factor authentication** — QR code enrollment via speakeasy, compatible with Google Authenticator and Authy
- **Bcrypt password hashing** — cost factor 12, salted, never stored in plain text
- **JWT session tokens** — signed with HMAC-SHA256, role-differentiated expiry, blacklisted on logout
- **Account lockout** — 5 consecutive failed logins triggers a 15-minute lock at the database level
- **Rate limiting** — per-email on login, per-user-ID on MFA verify
- **IDOR protection** — patients can only access their own data; ownership is derived from the JWT, never trusted from the client
- **Immutable prescriptions** — clinical fields (`diagnosis`, `medication`, `dosage`, `instructions`) are locked by a Mongoose pre-save hook after signing
- **Cryptographic prescription signatures** — SHA-256 hash of `doctorId:prescriptionId:timestamp`
- **PDF generation** — signed prescriptions downloadable as PDFs with embedded QR codes
- **Append-only audit log** — Mongoose hooks block all updates and deletes on the `AuditLog` collection
- **Helmet HTTP headers** — X-Frame-Options, X-Content-Type-Options, Strict-Transport-Security, and more
- **Token-based password reset** — short-lived JWT (15 min), user-enumeration-safe response

---

## Tech Stack

### Backend
| Package | Purpose |
|---|---|
| `express` | HTTP server and routing |
| `mongoose` | MongoDB ODM and schema enforcement |
| `bcryptjs` | Password hashing (cost 12) |
| `jsonwebtoken` | JWT signing and verification |
| `speakeasy` | TOTP/HOTP secret generation and verification |
| `qrcode` | QR code generation for MFA enrollment |
| `express-rate-limit` | Per-email and per-user rate limiting |
| `helmet` | Secure HTTP response headers |
| `pdfkit` | Prescription PDF generation |
| `morgan` | HTTP request logging |
| `dotenv` | Environment variable management |

### Frontend
| Package | Purpose |
|---|---|
| `react` 18 | UI framework |
| `vite` | Build tool and dev server |
| `tailwind css` | Utility-first styling |
| `react-router-dom` | Client-side routing with protected routes |
| `axios` | HTTP client with interceptors |
| `lucide-react` | Icon library |

---

## Project Structure

```
medreceipt/
├── server/
│   ├── server.js               # Express app entry point
│   ├── package.json
│   ├── .env.example
│   ├── seed.js                 # Base seed data (10 users, 15 prescriptions)
│   ├── seed-extra.js           # Additional seed data (3 patients, 8 prescriptions)
│   ├── config/
│   │   └── db.js               # MongoDB connection
│   ├── middleware/
│   │   ├── auth.js             # JWT verification + blacklist
│   │   └── rbac.js             # requireRole() middleware factory
│   ├── models/
│   │   ├── User.js             # User schema + toSafeJSON()
│   │   ├── Prescription.js     # Prescription schema + immutability hook
│   │   └── AuditLog.js         # Append-only audit log schema
│   ├── routes/
│   │   ├── auth.js             # Register, login, MFA, logout, password reset
│   │   ├── users.js            # User CRUD (role-scoped)
│   │   ├── prescriptions.js    # Prescription CRUD + PDF download
│   │   └── audit.js            # Audit log query endpoint
│   └── utils/
│       └── audit.js            # logEvent() helper
└── client/
    ├── index.html
    ├── vite.config.js
    ├── tailwind.config.js
    └── src/
        ├── main.jsx
        ├── App.jsx             # Route definitions
        ├── index.css
        ├── api/
        │   └── client.js       # Axios instance with auth interceptor
        ├── context/
        │   └── AuthContext.jsx # Global auth state
        ├── components/
        │   ├── Layout.jsx
        │   ├── ProtectedRoute.jsx
        │   ├── StatusBadge.jsx
        │   └── ui/Modal.jsx
        └── pages/
            ├── Login.jsx
            ├── Register.jsx
            ├── MfaVerify.jsx
            ├── ForgotPassword.jsx
            ├── ResetPassword.jsx
            ├── Profile.jsx           # Includes MFA setup + password change
            ├── AdminDashboard.jsx
            ├── AdminUsers.jsx
            ├── DoctorDashboard.jsx
            ├── DoctorPrescriptions.jsx
            ├── PatientDashboard.jsx
            ├── PatientsList.jsx
            ├── PatientDetail.jsx
            ├── PrescriptionDetail.jsx
            └── AuditLog.jsx
```

---

## Quick Start

### Prerequisites

- Node.js ≥ 18
- MongoDB (local instance or MongoDB Atlas)

### 1. Clone and install

```bash
git clone https://github.com/your-username/medreceipt.git
cd medreceipt
```

### 2. Set up the server

```bash
cd server
cp .env.example .env
# Edit .env — at minimum set JWT_SECRET and MONGO_URI
npm install
```

### 3. Seed the database

```bash
npm run seed          # base data: 10 users, 15 prescriptions
npm run seed:extra    # optional: 3 more patients, 8 more prescriptions
# or both:
npm run seed:all
```

> **Important:** When `seed.js` runs, it prints the TOTP base32 secret and current 6-digit code for MFA-enabled doctor accounts to the terminal. Save these to use with your authenticator app.

### 4. Start the server

```bash
npm run dev           # http://localhost:5000
```

### 5. Set up and start the client

```bash
cd ../client
npm install
npm run dev           # http://localhost:5173
```

---

## Environment Variables

Create `server/.env` from `server/.env.example`:

```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/medreceipt
JWT_SECRET=change-me-to-a-long-random-string-in-production
JWT_DOCTOR_EXPIRY=1h
JWT_PATIENT_EXPIRY=8h
JWT_PREAUTH_EXPIRY=5m
CLIENT_ORIGIN=http://localhost:5173
NODE_ENV=development
```

| Variable | Description |
|---|---|
| `JWT_SECRET` | Minimum 32 random characters. Used to sign all JWTs. |
| `JWT_DOCTOR_EXPIRY` | Access token lifetime for doctor accounts. |
| `JWT_PATIENT_EXPIRY` | Access token lifetime for patient accounts. |
| `JWT_PREAUTH_EXPIRY` | Lifetime of the pre-auth token issued during MFA login step 1. Default `5m`. |
| `CLIENT_ORIGIN` | CORS allowed origin. Set to your frontend URL in production. |
| `NODE_ENV` | Set to `production` to suppress dev-only password reset token leakage. |

---

## Seed Accounts

After `npm run seed`:

| Role | Email | Password | MFA |
|---|---|---|---|
| Admin | admin@medreceipt.io | Admin!Pass123 | — |
| Doctor | dr.house@medreceipt.io | Doctor!Pass123 | TOTP ✓ |
| Doctor | dr.wilson@medreceipt.io | Doctor!Pass123 | TOTP ✓ |
| Doctor | dr.cuddy@medreceipt.io | Doctor!Pass123 | — |
| Patient | alice@medreceipt.io | Patient!Pass123 | — |
| Patient | bob@medreceipt.io | Patient!Pass123 | — |
| Patient | carol@medreceipt.io | Patient!Pass123 | — |
| Patient | davis@medreceipt.io | Patient!Pass123 | — |
| Patient | emma@medreceipt.io | Patient!Pass123 | — |
| Patient | frank@medreceipt.io | Patient!Pass123 | — |

After `npm run seed:extra`, three additional patients are added: gundars, helena, ivars (same password pattern).

> For MFA-enabled doctors, use the TOTP secret printed to the terminal during seeding. Enter it manually in your authenticator app (Google Authenticator → Add account → Enter setup key).

---

## API Reference

All endpoints are prefixed with `/api/v1`.

### Authentication — `/auth`

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `POST` | `/auth/register` | Public | Patient self-registration |
| `POST` | `/auth/login` | Public | Login — returns token or `{mfaRequired, preauthToken}` |
| `POST` | `/auth/verify-mfa` | Public | Exchange preauth token + TOTP code for full access token |
| `POST` | `/auth/mfa/setup` | JWT | Generate TOTP secret and QR code (pending, not yet active) |
| `POST` | `/auth/mfa/confirm` | JWT | Verify pending TOTP code and activate MFA |
| `DELETE` | `/auth/mfa` | JWT | Disable MFA (requires current TOTP, or admin role) |
| `POST` | `/auth/change-password` | JWT | Change password (requires current password) |
| `POST` | `/auth/forgot-password` | Public | Request password reset token |
| `POST` | `/auth/reset-password` | Public | Submit reset token + new password |
| `POST` | `/auth/logout` | JWT | Blacklist current JWT by its `jti` |
| `GET` | `/auth/me` | JWT | Return current user (safe fields only) |

### Users — `/users`

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `GET` | `/users/me` | JWT | Current user profile |
| `PUT` | `/users/me` | JWT | Update own profile (name, phone, address, allergies, emergency contact) |
| `GET` | `/users` | Doctor / Admin | List users — doctors see patients only, admins see all; supports `search`, `role`, `isActive`, pagination |
| `GET` | `/users/:id` | Doctor / Admin | User detail with prescription counts |
| `POST` | `/users` | Admin | Provision a new user account |
| `PUT` | `/users/:id` | Admin | Update any user |
| `DELETE` | `/users/:id` | Admin | Soft-deactivate user (sets `isActive: false`) |

### Prescriptions — `/prescriptions`

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `GET` | `/prescriptions` | JWT | List prescriptions — scoped by role. Patients see only their own (IDOR enforced server-side). Supports `status`, `patientId`, pagination. |
| `POST` | `/prescriptions` | Doctor | Create a new prescription |
| `GET` | `/prescriptions/:id` | JWT | Prescription detail — IDOR enforced. Logs `PRESCRIPTION_VIEWED`. |
| `PUT` | `/prescriptions/:id` | Doctor | Update status or sign prescription. Clinical fields locked after signing. |
| `GET` | `/prescriptions/:id/pdf` | JWT | Download signed prescription as PDF with embedded QR code |

### Audit — `/audit`

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `GET` | `/audit` | Doctor / Admin | Query audit log. Supports `actorId`, `eventType`, `resourceId`, `success`, date range, pagination. |

---

## Security Architecture

### Password Hashing

Passwords are hashed with `bcryptjs` at cost factor 12 (2¹² = 4,096 iterations of blowfish). A random 22-character salt is generated per password and embedded in the hash string — no separate salt storage required. The plain password is discarded immediately and never persisted.

```
"MyPassword1!" → bcrypt.hash(password, 12) → "$2b$12$<salt><hash>"
```

At login, `bcrypt.compare()` performs a constant-time comparison — it takes the same amount of time whether the password is correct or not, which prevents timing attacks.

### JWT Tokens

After a successful login (and MFA if enabled), the server calls `issueToken()`:

```js
const jti = crypto.randomUUID();           // unique token ID for revocation
jwt.sign({ sub, role, email, jti }, JWT_SECRET, { expiresIn })
```

The token payload carries `sub` (user ID), `role`, `email`, and `jti`. Every protected route runs `verifyJWT` middleware which:

1. Extracts the `Bearer` token from the `Authorization` header
2. Verifies the signature against `JWT_SECRET`
3. Checks the `jti` is not in the blacklist
4. Rejects `type: 'preauth'` tokens (pre-MFA tokens cannot access protected routes)
5. Attaches `req.user` for downstream handlers

### MFA Login — Two-Token Flow

When a user with MFA enabled logs in with their password:

1. Server issues a **preauth token** (`type: 'preauth'`, 5 min expiry)
2. Client sends this token + 6-digit TOTP code to `POST /auth/verify-mfa`
3. Server verifies the TOTP via `speakeasy.totp.verify()` with `window: 2` (±60s clock tolerance)
4. On success, server issues the real access token

Preauth tokens are explicitly blocked in `verifyJWT` — they cannot be used to access any protected endpoint.

### JWT Blacklisting

On logout, the remaining lifetime of the token is calculated and the `jti` is stored in an in-memory `Map` with a TTL. Every call to `verifyJWT` checks this map. Expired entries are cleaned up automatically.

> In production, replace the in-memory map with Redis for persistence across server restarts and horizontal scaling.

### Rate Limiting

Two separate `express-rate-limit` instances protect the sensitive auth endpoints:

| Endpoint | Key | Limit | Window |
|---|---|---|---|
| `POST /auth/login` | Email address | 10 requests | 15 minutes |
| `POST /auth/verify-mfa` | User ID (from preauth token) | 10 requests | 10 minutes |

Keying by email (not IP) prevents shared-IP false positives and makes limits meaningful per account.

### Account Lockout

Independent of rate limiting, the `User` model tracks `failedLoginAttempts`. After 5 consecutive wrong passwords:

- `lockoutUntil` is set to `Date.now() + 15 minutes`
- `failedLoginAttempts` is reset to 0
- An `ACCOUNT_LOCKED` event is written to the audit log
- All subsequent login attempts return `423 Locked` with `retryAfter` in the response

A successful login resets both fields.

### IDOR Protection

Insecure Direct Object Reference protection is enforced server-side. The patient's identity is always derived from the verified JWT (`req.user.id`), never from query parameters or request body fields. Patients cannot request another patient's prescriptions — any attempt is blocked and logged as an `IDOR_ATTEMPT` event.

### Prescription Immutability

A Mongoose `pre('save')` hook enforces that the fields `diagnosis`, `medication`, `dosage`, and `instructions` cannot be modified once a prescription moves out of `DRAFT` status. Attempted modifications throw an error at the model layer before reaching the database.

Prescriptions are also cryptographically signed:

```js
SHA-256("doctorId:prescriptionId:timestamp") → signatureHash
```

This hash is stored on the prescription and embedded in the PDF, providing a tamper-evident audit trail.

### HTTP Security Headers

`helmet` adds the following headers to every response:

| Header | Protection |
|---|---|
| `X-Frame-Options` | Prevents clickjacking via iframes |
| `X-Content-Type-Options` | Prevents MIME-type sniffing |
| `Strict-Transport-Security` | Enforces HTTPS |
| `X-DNS-Prefetch-Control` | Controls DNS prefetching |
| `Referrer-Policy` | Limits referrer information leakage |

CORS is restricted to `CLIENT_ORIGIN` from the environment. Request bodies are capped at 1MB to prevent memory exhaustion.

### User Enumeration Prevention

`POST /auth/forgot-password` always returns the same response (`"If that email exists, a reset link has been sent."`) regardless of whether the email is registered. This prevents attackers from probing which addresses have accounts.

### Data Sanitisation

`User.toSafeJSON()` is called on every API response that includes a user object. It deletes the following fields before serialisation:

```
passwordHash · totpSecret · totpPending · failedLoginAttempts · lockoutUntil
```

These fields are never present in any API response, even in error conditions.

---

## Data Models

### User

```
fullName          String, required
email             String, unique, indexed
passwordHash      String (bcrypt $2b$12$…)
personalId        String, unique (national ID)
phone             String
role              DOCTOR | PATIENT | ADMIN
licenseNumber     String (doctors)
specialty         String (doctors)
department        String (doctors)
dateOfBirth       Date
address           String
emergencyContact  String
bloodType         A+|A-|B+|B-|AB+|AB-|O+|O-
allergies         String
totpSecret        String (active TOTP secret, never in API responses)
totpPending       String (unconfirmed secret during enrollment)
mfaEnabled        Boolean, default false
isVerified        Boolean
isActive          Boolean, default true
failedLoginAttempts Number, default 0
lockoutUntil      Date
lastLoginAt       Date
passwordChangedAt Date
```

### Prescription

```
patientId         ObjectId → User, indexed
doctorId          ObjectId → User, indexed
diagnosis         String, required
medication        String, required (locked after signing)
dosage            String, required (locked after signing)
instructions      String, required (locked after signing)
signatureHash     String (SHA-256 signature)
status            DRAFT | ACTIVE | DISPENSED | EXPIRED
expiresAt         Date
signedAt          Date
```

### AuditLog

```
actorId           ObjectId → User
actorEmail        String (denormalised — preserved if user is deleted)
eventType         String, indexed
resourceType      String
resourceId        String, indexed
ipAddress         String
userAgent         String
metadata          Object (arbitrary context)
success           Boolean, default true
createdAt         Date (auto)
```

**All `updateOne`, `findOneAndUpdate`, `deleteOne`, and `findOneAndDelete` operations on `AuditLog` throw an error at the Mongoose hook level.** The collection is append-only by design.

#### Audit event types

| Event | Trigger |
|---|---|
| `USER_REGISTERED` | New account created via `/auth/register` |
| `USER_LOGIN_SUCCESS` | Successful login (with or without MFA) |
| `USER_LOGIN_FAILED` | Wrong password |
| `ACCOUNT_LOCKED` | 5th consecutive failed login |
| `MFA_ENABLED` | TOTP confirmed and activated |
| `MFA_DISABLED` | MFA turned off |
| `MFA_FAILED` | Wrong TOTP code submitted |
| `USER_LOGOUT` | Token blacklisted |
| `PASSWORD_CHANGED` | Password updated via change-password |
| `PASSWORD_RESET_REQUESTED` | Forgot-password flow initiated |
| `PASSWORD_RESET_COMPLETED` | Password successfully reset |
| `USER_PROFILE_UPDATED` | Profile fields changed |
| `PRESCRIPTION_VIEWED` | Prescription detail accessed |
| `IDOR_ATTEMPT` | Patient tried to access another patient's record |

---

## Role Capabilities

| Action | Patient | Doctor | Admin |
|---|---|---|---|
| View own profile | ✓ | ✓ | ✓ |
| Edit own profile | ✓ | ✓ | ✓ |
| Enable / disable own MFA | ✓ | ✓ | ✓ |
| Change own password | ✓ | ✓ | ✓ |
| View own prescriptions | ✓ | — | — |
| View patient list | — | ✓ | ✓ |
| View any user's detail | — | ✓ | ✓ |
| Create prescription | — | ✓ | — |
| Sign / update prescription | — | ✓ | — |
| Download prescription PDF | ✓ | ✓ | ✓ |
| View audit log | — | ✓ (own) | ✓ (all) |
| Provision new users | — | — | ✓ |
| Edit any user | — | — | ✓ |
| Deactivate user | — | — | ✓ |
| Disable any user's MFA | — | — | ✓ |

---

## Two-Factor Authentication

### Enrollment

1. Log in and go to **Profile → Security**
2. Click **Enable two-factor authentication**
3. The server generates a random 160-bit secret via `speakeasy.generateSecret()` and stores it as `totpPending`
4. A QR code is rendered — scan it with Google Authenticator or Authy
5. Enter the 6-digit code from your app to confirm
6. On success, `totpSecret = totpPending`, `mfaEnabled = true`, `totpPending` is cleared

The two-step enrollment (setup → confirm) ensures the secret is never activated unless the user has successfully scanned and verified it.

### Login with MFA enabled

```
POST /auth/login     →  { mfaRequired: true, preauthToken: "..." }
POST /auth/verify-mfa (preauthToken + 6-digit code)  →  { token, user }
```

The preauth token is a short-lived JWT (default 5 minutes) that carries `type: 'preauth'`. It cannot be used to access any protected endpoint — `verifyJWT` explicitly rejects it. The full access token is only issued after the TOTP code is verified.

### Disabling MFA

`DELETE /auth/mfa` requires the current TOTP code (to prevent someone with a stolen session from disabling MFA). Admins can bypass this requirement.

---

## Password Policy

All passwords (registration, change, reset) must match:

```
/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]).{12,}$/
```

Requirements:
- Minimum **12 characters**
- At least one **uppercase** letter
- At least one **lowercase** letter
- At least one **digit**
- At least one **symbol** from the set `!@#$%^&*()_+-=[]{}...`

Passwords must also differ from the current password when changing.

---

## Audit Logging

The `logEvent()` utility writes to the `AuditLog` collection on every security-relevant action. It is wrapped in a `try/catch` — a logging failure is printed to `stderr` but never interrupts the request flow.

The `AuditLog` Mongoose schema has `pre` hooks on `updateOne`, `findOneAndUpdate`, `deleteOne`, and `findOneAndDelete` that throw unconditionally. Records can only be inserted, never modified or removed.

---

## Production Considerations

| Area | Current (dev) | Recommended for production |
|---|---|---|
| JWT blacklist | In-memory `Map` (lost on restart) | Redis with TTL |
| Password reset delivery | Token returned in API response | SMTP email with tokenised link |
| Content Security Policy | Disabled for SPA convenience | Configure per your frontend domains |
| HTTPS | Not enforced locally | Terminate TLS at load balancer; set `NODE_ENV=production` |
| MongoDB | Local instance | Atlas or managed cluster with authentication |
| `JWT_SECRET` | Placeholder string | Minimum 64 bytes of cryptographically random data (`openssl rand -hex 64`) |
| Rate limit storage | In-memory | Redis store via `rate-limit-redis` for multi-instance |
| Logs | `morgan` to stdout | Centralised log aggregation (Datadog, Cloudwatch, etc.) |
| TOTP window | 2 (±60s) | 1 (±30s) if clock sync can be guaranteed |

---

## License

MIT
