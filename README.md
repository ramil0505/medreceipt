# MedReceipt — Digital Prescription System

A full-stack clinical-grade prescription management system with role-based access control, two-factor authentication, immutable audit logging, and PDF generation.

## Stack

- **Frontend**: React 18 + Vite + Tailwind CSS
- **Backend**: Node.js + Express + Mongoose (MongoDB)
- **Auth**: JWT (short-lived) · bcrypt cost 12 · TOTP/HOTP via speakeasy
- **Security**: Rate-limiting · account lockout · JWT blacklist · IDOR protection · append-only audit log

## Roles

| Role    | Capabilities |
|---------|-------------|
| ADMIN   | Provision/deactivate users, view all audit logs, manage any account |
| DOCTOR  | Create/sign prescriptions, view patients, download PDFs |
| PATIENT | View own prescriptions, update profile, enable MFA |

---

## Quick Start

### Prerequisites
- Node.js ≥ 18
- MongoDB (local or Atlas)

### Server

```bash
cd server
cp .env.example .env          # edit JWT_SECRET and MONGO_URI
npm install
npm run seed                  # load base data (10 users, 15 prescriptions)
npm run seed:extra            # optional: 3 more patients, 8 more prescriptions
# or both at once:
npm run seed:all
npm run dev                   # http://localhost:5000
```

### Client

```bash
cd client
npm install
npm run dev                   # http://localhost:5173
```

---

## Seed Accounts

After running `npm run seed`:

| Role    | Email                       | Password        | MFA    |
|---------|-----------------------------|-----------------|--------|
| Admin   | admin@medreceipt.io         | Admin!Pass123   | —      |
| Doctor  | dr.house@medreceipt.io      | Doctor!Pass123  | TOTP ✓ |
| Doctor  | dr.wilson@medreceipt.io     | Doctor!Pass123  | TOTP ✓ |
| Doctor  | dr.cuddy@medreceipt.io      | Doctor!Pass123  | —      |
| Patient | alice@medreceipt.io         | Patient!Pass123 | —      |
| Patient | bob@medreceipt.io           | Patient!Pass123 | —      |
| Patient | carol@medreceipt.io         | Patient!Pass123 | —      |
| Patient | davis@medreceipt.io         | Patient!Pass123 | —      |
| Patient | emma@medreceipt.io          | Patient!Pass123 | —      |
| Patient | frank@medreceipt.io         | Patient!Pass123 | —      |

> **Note**: The TOTP base32 secret and current 6-digit code for MFA-enabled doctors are printed to the terminal when `seed.js` runs. Use an authenticator app (Google Authenticator, Authy) with the printed secret.

After running `npm run seed:extra`, three more patients are added: gundars, helena, ivars.

---

## Two-Factor Authentication

### What was fixed

The original project had a broken 2FA flow:
- **Missing MFA setup endpoint** — no way to enroll TOTP from the UI
- **Missing QR code generation** — users couldn't scan into an authenticator app
- **No confirm step** — secrets were saved without verification
- **No disable flow** — once set, MFA couldn't be removed
- **No password change** — profile had no way to rotate credentials
- **TOTP window too tight** — `window: 1` caused failures with slight clock skew

### Endpoints added

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/auth/mfa/setup` | Generate TOTP secret + QR code (pending) |
| `POST` | `/api/v1/auth/mfa/confirm` | Verify pending TOTP and activate MFA |
| `DELETE` | `/api/v1/auth/mfa` | Disable MFA (requires current TOTP) |
| `POST` | `/api/v1/auth/change-password` | Change password (requires current password) |

### How to enable MFA (UI)

1. Log in as any user
2. Go to **Profile**
3. In the **Security** section, click **Enable two-factor authentication**
4. Scan the QR code with your authenticator app
5. Enter the 6-digit code to confirm

---

## API Reference

### Auth

```
POST /api/v1/auth/register        Patient self-signup
POST /api/v1/auth/login           Returns token or {mfaRequired, preauthToken}
POST /api/v1/auth/verify-mfa      Exchange preauthToken + TOTP code for full token
POST /api/v1/auth/mfa/setup       Start MFA enrollment (returns QR code)
POST /api/v1/auth/mfa/confirm     Confirm TOTP and activate MFA
DELETE /api/v1/auth/mfa           Disable MFA
POST /api/v1/auth/change-password Change password
POST /api/v1/auth/logout          Blacklist JWT
GET  /api/v1/auth/me              Current user
```

### Users

```
GET    /api/v1/users              List users (doctor: patients only, admin: all)
GET    /api/v1/users/:id          User detail + rx counts
PUT    /api/v1/users/me           Update own profile
POST   /api/v1/users              Admin: provision new account
PUT    /api/v1/users/:id          Admin: update any user
DELETE /api/v1/users/:id          Admin: soft-deactivate user
```

### Prescriptions

```
GET    /api/v1/prescriptions      List (scoped by role)
POST   /api/v1/prescriptions      Doctor: create prescription
GET    /api/v1/prescriptions/:id  Detail (IDOR protected)
PUT    /api/v1/prescriptions/:id  Doctor: sign or update status
GET    /api/v1/prescriptions/:id/pdf  Download PDF
```

### Audit

```
GET /api/v1/audit                 Audit log (doctor/admin only, paginated)
```

---

## Security Design

- **JWT blacklisting** — in-memory on logout (Redis-ready)
- **Account lockout** — 5 failed logins → 15-minute lock
- **Rate limiting** — login: 5/15min, MFA verify: 10/10min
- **IDOR protection** — patients can only access their own prescriptions
- **Prescription immutability** — clinical fields locked after signing
- **Audit log** — append-only MongoDB collection (all writes blocked by Mongoose hooks)
- **TOTP window** — ±60 seconds (`window: 2`) to handle clock skew
- **totpPending** — 2-step MFA enrollment (setup → confirm) prevents invalid secrets being saved

## Environment Variables

```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/medreceipt
JWT_SECRET=<minimum 32 random chars>
JWT_DOCTOR_EXPIRY=1h
JWT_PATIENT_EXPIRY=8h
JWT_PREAUTH_EXPIRY=5m
CLIENT_ORIGIN=http://localhost:5173
NODE_ENV=development
```
