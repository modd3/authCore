# AuthCore — Pluggable Auth Microservice
### Project Specification & Implementation Guide

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Tech Stack & Requirements](#3-tech-stack--requirements)
4. [Project Structure](#4-project-structure)
5. [Database Schema](#5-database-schema)
6. [Environment Configuration](#6-environment-configuration)
7. [API Reference](#7-api-reference)
8. [Implementation Phases](#8-implementation-phases)
9. [Core Logic & Code Examples](#9-core-logic--code-examples)
10. [Security Requirements](#10-security-requirements)
11. [Client SDK](#11-client-sdk)
12. [Admin Dashboard](#12-admin-dashboard)
13. [Testing](#13-testing)
14. [Error Handling Standard](#14-error-handling-standard)
15. [Deployment](#15-deployment)
16. [Open Source Checklist](#16-open-source-checklist)

---

## 1. Project Overview

**AuthCore** is a self-hosted, language-agnostic authentication microservice. Any application — regardless of language or framework — integrates auth by making HTTP calls to the AuthCore server. No auth logic is written in the consuming app.

### Goals
- Drop-in auth for any project with minimal configuration
- Security-first defaults — misconfiguration is loud, not silent
- Clean, well-documented codebase suitable for open source
- Demonstrates fullstack + cybersecurity thinking in one project

### What it is NOT
- A managed auth service (no cloud dependency)
- A framework plugin (not tied to Express, Next.js, Django, etc.)
- Feature-complete on day one — ship a clean MVP first

### MVP Feature Set
| Feature | Status |
|---|---|
| Email/password registration | MVP |
| Login with JWT (access + refresh tokens) | MVP |
| Rotating refresh tokens | MVP |
| Logout (single device + all devices) | MVP |
| Password reset via email | MVP |
| Rate limiting & brute-force protection | MVP |
| Token verification endpoint | MVP |
| PostgreSQL adapter | MVP |
| JS/TS client SDK | MVP |
| Admin dashboard (basic) | MVP |
| Google OAuth | v2 |
| GitHub OAuth | v2 |
| TOTP / 2FA | v2 |
| Magic link login | v2 |
| Multi-tenancy | v2 |
| MongoDB adapter | v2 |

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Your Application                   │
│  (Node, Python, Go, PHP — any language)             │
│                                                     │
│   Uses AuthCore JS SDK  ──OR──  raw HTTP calls      │
└──────────────────┬──────────────────────────────────┘
                   │ HTTP / REST
                   ▼
┌─────────────────────────────────────────────────────┐
│               AuthCore Server                       │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  Routes  │→ │  Guards  │→ │  Service Layer   │  │
│  └──────────┘  └──────────┘  └────────┬─────────┘  │
│                                       │             │
│  ┌────────────────────────────────────▼──────────┐  │
│  │            Adapter Layer                      │  │
│  │  ┌─────────────┐  ┌──────────┐  ┌─────────┐  │  │
│  │  │  DB Adapter │  │  Email   │  │  Cache  │  │  │
│  │  │ (Postgres)  │  │ Adapter  │  │ (Redis) │  │  │
│  │  └─────────────┘  └──────────┘  └─────────┘  │  │
│  └───────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┬──┘
                                                   │
            ┌──────────────────────────────────────▼─┐
            │           Webhook Dispatcher            │
            │  Fires events to your app on:           │
            │  user.registered / login.failed /       │
            │  password.reset / session.revoked       │
            └────────────────────────────────────────┘
```

### Request Flow — Login Example
```
1. App sends POST /auth/login  { email, password }
2. Rate limiter checks IP — blocks if threshold exceeded
3. Service layer fetches user from DB
4. Argon2 verifies password hash
5. Issues access token (15 min) + refresh token (7 days)
6. Refresh token stored (hashed) in DB
7. Webhook fires: login.success event to app
8. Returns: { accessToken, refreshToken, user }
```

---

## 3. Tech Stack & Requirements

### Runtime Requirements
```
Node.js        >= 18.0.0
PostgreSQL     >= 14
Redis          >= 6 (for rate limiting + token blacklist)
npm            >= 9
```

### Server Dependencies
```json
{
  "dependencies": {
    "fastify": "^4.26.0",
    "@fastify/cors": "^9.0.1",
    "@fastify/rate-limit": "^9.1.0",
    "@fastify/swagger": "^8.14.0",
    "argon2": "^0.31.2",
    "jsonwebtoken": "^9.0.2",
    "zod": "^3.22.4",
    "nodemailer": "^6.9.9",
    "pg": "^8.11.3",
    "ioredis": "^5.3.2",
    "uuid": "^9.0.1",
    "dotenv": "^16.4.1",
    "winston": "^3.11.0",
    "axios": "^1.6.7"
  },
  "devDependencies": {
    "typescript": "^5.3.3",
    "@types/node": "^20.11.5",
    "@types/jsonwebtoken": "^9.0.5",
    "@types/nodemailer": "^6.4.14",
    "vitest": "^1.2.2",
    "supertest": "^6.3.4",
    "@types/supertest": "^6.0.2",
    "tsx": "^4.7.0",
    "eslint": "^8.56.0"
  }
}
```

> **Why Fastify over Express?**
> Fastify is 2–3x faster, has built-in schema validation, TypeScript support out of the box, and a plugin architecture that maps well to this project's adapter pattern.

> **Why Argon2 over bcrypt?**
> Argon2id is the current OWASP recommendation for password hashing. It's resistant to GPU and side-channel attacks. bcrypt is acceptable but outdated.

---

## 4. Project Structure

```
authcore/
├── src/
│   ├── server.ts               # Fastify instance, plugin registration
│   ├── app.ts                  # Entry point
│   │
│   ├── config/
│   │   ├── index.ts            # Config loader (validates env vars on startup)
│   │   └── defaults.ts         # Secure default values
│   │
│   ├── routes/
│   │   ├── auth.routes.ts      # /auth/* endpoints
│   │   ├── token.routes.ts     # /token/* endpoints
│   │   └── admin.routes.ts     # /admin/* endpoints (protected)
│   │
│   ├── services/
│   │   ├── auth.service.ts     # Registration, login, logout logic
│   │   ├── token.service.ts    # JWT creation, rotation, verification
│   │   ├── email.service.ts    # Password reset emails
│   │   └── webhook.service.ts  # Outbound webhook dispatcher
│   │
│   ├── adapters/
│   │   ├── db/
│   │   │   ├── interface.ts    # IDatabase interface (all adapters implement this)
│   │   │   └── postgres.ts     # PostgreSQL adapter
│   │   ├── cache/
│   │   │   ├── interface.ts    # ICache interface
│   │   │   └── redis.ts        # Redis adapter
│   │   └── email/
│   │       ├── interface.ts    # IEmail interface
│   │       └── nodemailer.ts   # Nodemailer adapter
│   │
│   ├── middleware/
│   │   ├── auth.guard.ts       # Validates access tokens on protected routes
│   │   ├── admin.guard.ts      # Validates admin-level tokens
│   │   └── validate.ts         # Zod schema validation middleware
│   │
│   ├── schemas/
│   │   ├── auth.schema.ts      # Zod schemas for auth request bodies
│   │   └── token.schema.ts     # Zod schemas for token requests
│   │
│   ├── utils/
│   │   ├── crypto.ts           # Token generation helpers
│   │   ├── errors.ts           # Custom error classes
│   │   └── logger.ts           # Winston logger setup
│   │
│   └── types/
│       └── index.ts            # Global TypeScript types
│
├── dashboard/                  # React admin UI (separate Vite app)
│   ├── src/
│   │   ├── App.tsx
│   │   ├── pages/
│   │   │   ├── Users.tsx
│   │   │   ├── Sessions.tsx
│   │   │   └── LoginAttempts.tsx
│   │   └── components/
│   └── package.json
│
├── sdk/                        # JS/TS client SDK
│   ├── src/
│   │   └── index.ts
│   └── package.json
│
├── tests/
│   ├── unit/
│   │   ├── auth.service.test.ts
│   │   └── token.service.test.ts
│   └── integration/
│       └── auth.routes.test.ts
│
├── migrations/
│   ├── 001_create_users.sql
│   ├── 002_create_sessions.sql
│   ├── 003_create_refresh_tokens.sql
│   └── 004_create_login_attempts.sql
│
├── .env.example
├── auth.config.example.yaml
├── docker-compose.yml
├── Dockerfile
├── package.json
├── tsconfig.json
└── README.md
```

---

## 5. Database Schema

### `users`
```sql
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         VARCHAR(255) UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  is_verified   BOOLEAN DEFAULT FALSE,
  is_active     BOOLEAN DEFAULT TRUE,
  role          VARCHAR(50) DEFAULT 'user',
  metadata      JSONB DEFAULT '{}',
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

### `refresh_tokens`
```sql
CREATE TABLE refresh_tokens (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash    TEXT NOT NULL,           -- store hash, never plain token
  family        UUID NOT NULL,           -- token family for rotation detection
  device_info   JSONB DEFAULT '{}',      -- optional: user-agent, IP
  expires_at    TIMESTAMPTZ NOT NULL,
  revoked       BOOLEAN DEFAULT FALSE,
  revoked_at    TIMESTAMPTZ,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_family  ON refresh_tokens(family);
```

> **Token families explained:** Each initial login creates a new `family` UUID. Every refresh creates a new token in the same family. If a token from an old family member is used (reuse detected), the entire family is revoked — all sessions for that user from that login are killed. This detects token theft.

### `password_reset_tokens`
```sql
CREATE TABLE password_reset_tokens (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash TEXT NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  used       BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### `login_attempts`
```sql
CREATE TABLE login_attempts (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email       VARCHAR(255),
  ip_address  INET NOT NULL,
  success     BOOLEAN NOT NULL,
  reason      VARCHAR(100),             -- 'invalid_password', 'user_not_found', etc.
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_login_attempts_ip    ON login_attempts(ip_address, created_at);
CREATE INDEX idx_login_attempts_email ON login_attempts(email, created_at);
```

### `webhook_configs`
```sql
CREATE TABLE webhook_configs (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  url        TEXT NOT NULL,
  events     TEXT[] NOT NULL,           -- ['user.registered', 'login.failed']
  secret     TEXT NOT NULL,             -- HMAC signing secret
  is_active  BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 6. Environment Configuration

### `.env.example`
```bash
# ── Server ──────────────────────────────────────────
PORT=4000
NODE_ENV=development               # development | production
ALLOWED_ORIGINS=http://localhost:3000

# ── Database ────────────────────────────────────────
DATABASE_URL=postgresql://user:password@localhost:5432/authcore

# ── Redis ───────────────────────────────────────────
REDIS_URL=redis://localhost:6379

# ── JWT ─────────────────────────────────────────────
# REQUIRED: min 32 chars in production, server refuses to start otherwise
JWT_ACCESS_SECRET=change-this-to-a-long-random-string-in-production
JWT_REFRESH_SECRET=change-this-to-a-different-long-random-string
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

# ── Email ────────────────────────────────────────────
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your@email.com
SMTP_PASS=your-app-password
EMAIL_FROM="AuthCore <noreply@yourdomain.com>"

# ── Admin ────────────────────────────────────────────
ADMIN_SECRET=change-this-secret      # Used to issue admin tokens
```

### Config Validation (runs on startup)
```typescript
// src/config/index.ts
import { z } from 'zod'

const configSchema = z.object({
  port: z.number().default(4000),
  nodeEnv: z.enum(['development', 'production', 'test']),
  databaseUrl: z.string().url(),
  redisUrl: z.string(),
  jwt: z.object({
    accessSecret: z.string().min(32, 'JWT_ACCESS_SECRET must be at least 32 characters'),
    refreshSecret: z.string().min(32, 'JWT_REFRESH_SECRET must be at least 32 characters'),
    accessExpiry: z.string().default('15m'),
    refreshExpiry: z.string().default('7d'),
  }),
  email: z.object({
    host: z.string(),
    port: z.number(),
    user: z.string().email(),
    pass: z.string(),
    from: z.string(),
  }),
  adminSecret: z.string().min(32, 'ADMIN_SECRET must be at least 32 characters'),
})

export function loadConfig() {
  const result = configSchema.safeParse({
    port: Number(process.env.PORT),
    nodeEnv: process.env.NODE_ENV,
    databaseUrl: process.env.DATABASE_URL,
    redisUrl: process.env.REDIS_URL,
    jwt: {
      accessSecret: process.env.JWT_ACCESS_SECRET,
      refreshSecret: process.env.JWT_REFRESH_SECRET,
      accessExpiry: process.env.JWT_ACCESS_EXPIRY,
      refreshExpiry: process.env.JWT_REFRESH_EXPIRY,
    },
    email: {
      host: process.env.SMTP_HOST,
      port: Number(process.env.SMTP_PORT),
      user: process.env.SMTP_USER,
      pass: process.env.SMTP_PASS,
      from: process.env.EMAIL_FROM,
    },
    adminSecret: process.env.ADMIN_SECRET,
  })

  if (!result.success) {
    console.error('❌ Invalid configuration:')
    result.error.errors.forEach(e => console.error(`  ${e.path.join('.')}: ${e.message}`))
    process.exit(1)   // Hard stop — never run with bad config
  }

  return result.data
}
```

---

## 7. API Reference

### Base URL
```
http://localhost:4000
```

### Authentication Header (for protected routes)
```
Authorization: Bearer <accessToken>
```

---

### POST `/auth/register`
Register a new user.

**Request**
```json
{
  "email": "user@example.com",
  "password": "MyStr0ngP@ss!"
}
```

**Password Rules (enforced server-side)**
- Minimum 8 characters
- At least one uppercase letter
- At least one number
- At least one special character

**Response `201`**
```json
{
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "role": "user",
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

**Response `409` — email already registered**
```json
{
  "error": "CONFLICT",
  "message": "An account with this email already exists"
}
```

---

### POST `/auth/login`
Authenticate a user and issue tokens.

**Request**
```json
{
  "email": "user@example.com",
  "password": "MyStr0ngP@ss!"
}
```

**Response `200`**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 900,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "role": "user"
  }
}
```

**Response `401`**
```json
{
  "error": "UNAUTHORIZED",
  "message": "Invalid credentials"
}
```

**Response `429` — rate limited**
```json
{
  "error": "TOO_MANY_REQUESTS",
  "message": "Too many login attempts. Try again in 15 minutes.",
  "retryAfter": 900
}
```

> **Security note:** Always return the same error message for "wrong password" and "user not found". Different messages reveal whether an email is registered (user enumeration attack).

---

### POST `/auth/logout`
Revoke the current refresh token.

**Request** *(requires Authorization header)*
```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response `200`**
```json
{ "message": "Logged out successfully" }
```

---

### POST `/auth/logout-all`
Revoke all refresh tokens for the authenticated user (all devices).

**Request** *(requires Authorization header — no body)*

**Response `200`**
```json
{ "message": "All sessions revoked" }
```

---

### POST `/token/refresh`
Exchange a refresh token for a new access + refresh token pair.

**Request**
```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response `200`**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 900
}
```

**Response `401` — reuse detected (token theft)**
```json
{
  "error": "TOKEN_REUSE_DETECTED",
  "message": "All sessions have been revoked due to suspicious activity"
}
```

---

### POST `/token/verify`
Verify an access token and return the decoded payload. Your app calls this to validate a token without needing the JWT secret.

**Request**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response `200`**
```json
{
  "valid": true,
  "payload": {
    "sub": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "role": "user",
    "iat": 1705312200,
    "exp": 1705313100
  }
}
```

**Response `401`**
```json
{
  "valid": false,
  "error": "TOKEN_EXPIRED"
}
```

---

### POST `/auth/forgot-password`
Trigger a password reset email.

**Request**
```json
{
  "email": "user@example.com"
}
```

**Response `200`** *(always 200, even if email not found — prevents enumeration)*
```json
{
  "message": "If an account with that email exists, a reset link has been sent."
}
```

---

### POST `/auth/reset-password`
Complete a password reset.

**Request**
```json
{
  "token": "reset-token-from-email",
  "password": "MyNewStr0ngP@ss!"
}
```

**Response `200`**
```json
{ "message": "Password updated successfully" }
```

---

### GET `/auth/me`
Get the authenticated user's profile. *(requires Authorization header)*

**Response `200`**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "role": "user",
  "isVerified": true,
  "createdAt": "2024-01-15T10:30:00Z"
}
```

---

### Admin Routes *(require admin token)*

| Method | Path | Description |
|---|---|---|
| `GET` | `/admin/users` | List all users (paginated) |
| `GET` | `/admin/users/:id` | Get single user |
| `PATCH` | `/admin/users/:id` | Update user (role, active status) |
| `DELETE` | `/admin/users/:id` | Delete user |
| `GET` | `/admin/sessions` | List active sessions |
| `DELETE` | `/admin/sessions/:userId` | Revoke all sessions for user |
| `GET` | `/admin/login-attempts` | View login attempt logs |
| `GET` | `/admin/stats` | Usage stats |

---

## 8. Implementation Phases

### Phase 1 — Foundation (Week 1–2)
**Goal:** Server boots, connects to DB, one working endpoint.

- [ ] Initialize TypeScript project (`tsconfig.json`, linting)
- [ ] Set up Fastify server with CORS and basic plugin structure
- [ ] Implement config loader with validation (fail loudly on bad config)
- [ ] Write PostgreSQL adapter and connection pool
- [ ] Write and run all SQL migrations
- [ ] Implement `POST /auth/register` end-to-end
- [ ] Set up Winston logging (request logs, error logs, separate files)
- [ ] Write first unit tests (config validation, password schema)

**Checkpoint:** `curl -X POST localhost:4000/auth/register` registers a user in the DB.

---

### Phase 2 — Auth Core (Week 3–4)
**Goal:** Full login/logout/token cycle working.

- [ ] Implement token service (sign, verify, access + refresh)
- [ ] Implement `POST /auth/login` with Argon2 verification
- [ ] Implement refresh token storage (hashed) and retrieval
- [ ] Implement `POST /token/refresh` with rotation logic
- [ ] Implement `POST /token/verify`
- [ ] Implement `POST /auth/logout` and `POST /auth/logout-all`
- [ ] Implement `GET /auth/me` with auth guard middleware
- [ ] Add Redis adapter for token blacklist
- [ ] Write integration tests for the full auth flow

**Checkpoint:** Full login → refresh → logout cycle works end-to-end.

---

### Phase 3 — Security Hardening (Week 5)
**Goal:** Production-safe security features in place.

- [ ] Add rate limiting (Redis-backed, per-IP and per-email)
- [ ] Implement login attempt logging
- [ ] Implement account lockout after N failed attempts
- [ ] Add password reset flow (token generation → email → reset)
- [ ] Set up Nodemailer email adapter
- [ ] Add `Helmet`-equivalent security headers in Fastify
- [ ] Implement token reuse detection (family invalidation)
- [ ] Add request input sanitization

**Checkpoint:** Brute-force protection tested, password reset email works.

---

### Phase 4 — Webhooks & Admin (Week 6–7)
**Goal:** Extensibility and observability.

- [ ] Implement webhook dispatcher service
- [ ] Add HMAC signing to webhook payloads
- [ ] Implement retry logic for failed webhook deliveries
- [ ] Implement all admin API routes with admin guard
- [ ] Build admin dashboard React app (Vite + React)
- [ ] Dashboard: user list with search/filter, session viewer, log viewer
- [ ] Serve dashboard as static files from the auth server

**Checkpoint:** App receives webhook events; admin dashboard shows users/sessions.

---

### Phase 5 — SDK & Documentation (Week 8)
**Goal:** Developer experience and portfolio readiness.

- [ ] Build JS/TS SDK (see Section 11)
- [ ] Write full API documentation (Swagger/OpenAPI auto-generated)
- [ ] Write README with quickstart guide
- [ ] Write architecture decision records (ADRs)
- [ ] Write security audit document
- [ ] Add Dockerfile and docker-compose.yml
- [ ] Run OWASP ZAP scan against the server, document findings

**Checkpoint:** A fresh developer can clone, `docker-compose up`, and have a running auth server in under 5 minutes.

---

## 9. Core Logic & Code Examples

### Password Hashing (`src/utils/crypto.ts`)
```typescript
import argon2 from 'argon2'

// OWASP recommended Argon2id config (2024)
const ARGON2_OPTIONS = {
  type: argon2.argon2id,
  memoryCost: 65536,   // 64 MB
  timeCost: 3,
  parallelism: 4,
}

export async function hashPassword(password: string): Promise<string> {
  return argon2.hash(password, ARGON2_OPTIONS)
}

export async function verifyPassword(hash: string, password: string): Promise<boolean> {
  return argon2.verify(hash, password)
}

export function generateSecureToken(): string {
  // 32 bytes = 256 bits of entropy, URL-safe base64
  return require('crypto').randomBytes(32).toString('base64url')
}
```

---

### Token Service (`src/services/token.service.ts`)
```typescript
import jwt from 'jsonwebtoken'
import { createHash } from 'crypto'
import { v4 as uuidv4 } from 'uuid'
import { loadConfig } from '../config/index.js'
import type { IDatabase } from '../adapters/db/interface.js'

const config = loadConfig()

export interface TokenPayload {
  sub: string       // user ID
  email: string
  role: string
}

export function signAccessToken(payload: TokenPayload): string {
  return jwt.sign(payload, config.jwt.accessSecret, {
    expiresIn: config.jwt.accessExpiry,
    issuer: 'authcore',
  })
}

export function verifyAccessToken(token: string): TokenPayload {
  return jwt.verify(token, config.jwt.accessSecret, {
    issuer: 'authcore',
  }) as TokenPayload
}

export async function issueRefreshToken(
  userId: string,
  db: IDatabase,
  family?: string   // omit to start a new family (fresh login)
): Promise<string> {
  const rawToken = require('crypto').randomBytes(48).toString('base64url')
  const tokenHash = createHash('sha256').update(rawToken).digest('hex')
  const tokenFamily = family ?? uuidv4()

  const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)

  await db.query(
    `INSERT INTO refresh_tokens (user_id, token_hash, family, expires_at)
     VALUES ($1, $2, $3, $4)`,
    [userId, tokenHash, tokenFamily, expiresAt]
  )

  // Return raw token (never stored, only the hash is)
  return rawToken
}

export async function rotateRefreshToken(
  rawToken: string,
  db: IDatabase
): Promise<{ accessToken: string; refreshToken: string } | null> {
  const tokenHash = createHash('sha256').update(rawToken).digest('hex')

  const existing = await db.queryOne(
    `SELECT rt.*, u.email, u.role
     FROM refresh_tokens rt
     JOIN users u ON u.id = rt.user_id
     WHERE rt.token_hash = $1`,
    [tokenHash]
  )

  if (!existing) return null

  // Token reuse detected — entire family is compromised
  if (existing.revoked) {
    await db.query(
      `UPDATE refresh_tokens SET revoked = TRUE, revoked_at = NOW()
       WHERE family = $1`,
      [existing.family]
    )
    throw new Error('TOKEN_REUSE_DETECTED')
  }

  // Check expiry
  if (new Date(existing.expires_at) < new Date()) {
    return null
  }

  // Revoke current token
  await db.query(
    `UPDATE refresh_tokens SET revoked = TRUE, revoked_at = NOW()
     WHERE token_hash = $1`,
    [tokenHash]
  )

  // Issue new tokens (same family)
  const payload: TokenPayload = {
    sub: existing.user_id,
    email: existing.email,
    role: existing.role,
  }

  const accessToken = signAccessToken(payload)
  const refreshToken = await issueRefreshToken(existing.user_id, db, existing.family)

  return { accessToken, refreshToken }
}
```

---

### Auth Service — Login (`src/services/auth.service.ts`)
```typescript
import { hashPassword, verifyPassword } from '../utils/crypto.js'
import { signAccessToken, issueRefreshToken } from './token.service.js'
import { AppError } from '../utils/errors.js'
import type { IDatabase } from '../adapters/db/interface.js'
import type { ICache } from '../adapters/cache/interface.js'

// How many failed attempts before lockout
const MAX_ATTEMPTS = 5
const LOCKOUT_DURATION = 15 * 60  // 15 minutes in seconds

export async function loginUser(
  email: string,
  password: string,
  ip: string,
  db: IDatabase,
  cache: ICache
) {
  // 1. Check lockout (Redis key: lockout:{ip})
  const lockoutKey = `lockout:${ip}`
  const isLocked = await cache.get(lockoutKey)
  if (isLocked) {
    throw new AppError('TOO_MANY_REQUESTS', 'Too many attempts. Try again later.', 429)
  }

  // 2. Fetch user — ALWAYS compare password even if user not found
  //    This prevents timing attacks revealing valid emails
  const user = await db.queryOne(
    'SELECT * FROM users WHERE email = $1 AND is_active = TRUE',
    [email]
  )

  // Dummy hash to compare against if user not found (constant time)
  const DUMMY_HASH = '$argon2id$v=19$m=65536,t=3,p=4$dummyhash'
  const hashToVerify = user?.password_hash ?? DUMMY_HASH

  const passwordValid = await verifyPassword(hashToVerify, password)

  if (!user || !passwordValid) {
    // 3. Log failed attempt
    await db.query(
      `INSERT INTO login_attempts (email, ip_address, success, reason)
       VALUES ($1, $2, FALSE, $3)`,
      [email, ip, !user ? 'user_not_found' : 'invalid_password']
    )

    // 4. Increment attempt counter, lock if threshold reached
    const attemptsKey = `attempts:${ip}`
    const attempts = await cache.increment(attemptsKey)
    if (attempts === 1) await cache.expire(attemptsKey, LOCKOUT_DURATION)
    if (attempts >= MAX_ATTEMPTS) {
      await cache.set(lockoutKey, '1', LOCKOUT_DURATION)
    }

    // 5. Same error message regardless of reason (prevent enumeration)
    throw new AppError('UNAUTHORIZED', 'Invalid credentials', 401)
  }

  // 6. Success — clear attempt counter
  await cache.del(`attempts:${ip}`)

  await db.query(
    `INSERT INTO login_attempts (email, ip_address, success)
     VALUES ($1, $2, TRUE)`,
    [email, ip]
  )

  // 7. Issue tokens
  const accessToken = signAccessToken({ sub: user.id, email: user.email, role: user.role })
  const refreshToken = await issueRefreshToken(user.id, db)

  return {
    accessToken,
    refreshToken,
    expiresIn: 900,
    user: { id: user.id, email: user.email, role: user.role },
  }
}
```

---

### Auth Guard Middleware (`src/middleware/auth.guard.ts`)
```typescript
import { FastifyRequest, FastifyReply } from 'fastify'
import { verifyAccessToken } from '../services/token.service.js'
import { AppError } from '../utils/errors.js'

export async function authGuard(request: FastifyRequest, reply: FastifyReply) {
  const authHeader = request.headers.authorization
  if (!authHeader?.startsWith('Bearer ')) {
    return reply.status(401).send({ error: 'UNAUTHORIZED', message: 'Missing token' })
  }

  const token = authHeader.slice(7)

  try {
    const payload = verifyAccessToken(token)
    request.user = payload   // attach to request for downstream handlers
  } catch (err: any) {
    const code = err.name === 'TokenExpiredError' ? 'TOKEN_EXPIRED' : 'INVALID_TOKEN'
    return reply.status(401).send({ error: code, message: 'Invalid or expired token' })
  }
}
```

---

### Webhook Service (`src/services/webhook.service.ts`)
```typescript
import axios from 'axios'
import { createHmac } from 'crypto'
import type { IDatabase } from '../adapters/db/interface.js'

export type WebhookEvent =
  | 'user.registered'
  | 'login.success'
  | 'login.failed'
  | 'password.reset'
  | 'session.revoked'
  | 'account.locked'

export async function dispatchWebhook(
  event: WebhookEvent,
  payload: Record<string, unknown>,
  db: IDatabase
) {
  const hooks = await db.query(
    `SELECT * FROM webhook_configs
     WHERE is_active = TRUE AND $1 = ANY(events)`,
    [event]
  )

  const timestamp = Date.now()
  const body = JSON.stringify({ event, payload, timestamp })

  for (const hook of hooks) {
    const signature = createHmac('sha256', hook.secret)
      .update(body)
      .digest('hex')

    // Fire and retry up to 3 times — don't await (non-blocking)
    sendWithRetry(hook.url, body, signature).catch(err =>
      console.error(`Webhook delivery failed for ${hook.url}:`, err.message)
    )
  }
}

async function sendWithRetry(url: string, body: string, signature: string, attempt = 1) {
  try {
    await axios.post(url, body, {
      headers: {
        'Content-Type': 'application/json',
        'X-AuthCore-Signature': `sha256=${signature}`,
        'X-AuthCore-Timestamp': String(Date.now()),
      },
      timeout: 5000,
    })
  } catch (err) {
    if (attempt < 3) {
      // Exponential backoff: 1s, 2s, 4s
      await new Promise(r => setTimeout(r, 1000 * Math.pow(2, attempt - 1)))
      return sendWithRetry(url, body, signature, attempt + 1)
    }
    throw err
  }
}
```

---

## 10. Security Requirements

### Mandatory (ship before MVP)
- [ ] Passwords hashed with **Argon2id** (never MD5, SHA-1, or plain bcrypt)
- [ ] JWT secrets minimum 32 characters — server refuses to start otherwise
- [ ] Refresh tokens stored as **SHA-256 hash only** — raw token never persisted
- [ ] Refresh token rotation on every use
- [ ] Token reuse detection with full family revocation
- [ ] Rate limiting: 5 failed logins per IP per 15 minutes
- [ ] Same error message for "wrong password" and "user not found"
- [ ] Same response time for failed logins regardless of reason (prevent timing attacks)
- [ ] Password reset tokens expire in 1 hour, single use only
- [ ] HTTPS enforced in production mode (config validator rejects HTTP origins)
- [ ] All user input validated with Zod before reaching service layer
- [ ] SQL parameterized queries only (no string concatenation)
- [ ] Security headers: `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`

### Threat Model (document these)
| Threat | Mitigation |
|---|---|
| Brute force login | Rate limiting + account lockout |
| Credential stuffing | Same — rate limit by IP |
| Token theft (refresh) | Token rotation + reuse detection |
| User enumeration | Constant-time comparison + same error messages |
| Password reset abuse | Short expiry + single use + no enumeration |
| SQL injection | Parameterized queries only |
| XSS via auth responses | JSON API only, no HTML rendering |
| MITM | HTTPS enforced in production |
| Weak secrets | Config validation on startup |

### Self-Audit Checklist (run before open source release)
- [ ] Run OWASP ZAP scan — document and remediate all findings
- [ ] Run `npm audit` — resolve high/critical vulnerabilities
- [ ] Check all endpoints against OWASP Top 10 (at minimum: A01–A05)
- [ ] Verify no secrets in Git history (`git log -p | grep -i secret`)
- [ ] Verify `.env` is in `.gitignore`
- [ ] Test rate limiting manually with a script
- [ ] Test token reuse detection manually
- [ ] Test password reset token expiry
- [ ] Attempt SQL injection on all user-input fields

---

## 11. Client SDK

The SDK is a thin TypeScript wrapper. It handles token storage, auto-refresh on 401, and typed responses. No auth logic lives here.

### Usage (developer's perspective)
```typescript
import { AuthCore } from '@authcore/sdk'

const auth = new AuthCore({ baseUrl: 'http://localhost:4000' })

// Register
const user = await auth.register('user@example.com', 'password123')

// Login
const session = await auth.login('user@example.com', 'password123')

// Get current user (auto-attaches Bearer token)
const me = await auth.getUser()

// Refresh token (called automatically on 401)
await auth.refresh()

// Logout
await auth.logout()
```

### SDK Implementation (`sdk/src/index.ts`)
```typescript
interface AuthCoreConfig {
  baseUrl: string
  storage?: 'memory' | 'localStorage'  // default: memory (safer)
}

interface Session {
  accessToken: string
  refreshToken: string
  expiresIn: number
  user: { id: string; email: string; role: string }
}

export class AuthCore {
  private baseUrl: string
  private accessToken: string | null = null
  private refreshToken: string | null = null

  constructor(config: AuthCoreConfig) {
    this.baseUrl = config.baseUrl.replace(/\/$/, '')
  }

  async register(email: string, password: string) {
    return this.request('/auth/register', 'POST', { email, password })
  }

  async login(email: string, password: string): Promise<Session> {
    const session = await this.request<Session>('/auth/login', 'POST', { email, password })
    this.accessToken = session.accessToken
    this.refreshToken = session.refreshToken
    return session
  }

  async logout() {
    await this.request('/auth/logout', 'POST', { refreshToken: this.refreshToken }, true)
    this.accessToken = null
    this.refreshToken = null
  }

  async getUser() {
    return this.request('/auth/me', 'GET', null, true)
  }

  async refresh() {
    const result = await this.request<Session>('/token/refresh', 'POST', {
      refreshToken: this.refreshToken,
    })
    this.accessToken = result.accessToken
    this.refreshToken = result.refreshToken
    return result
  }

  private async request<T = any>(
    path: string,
    method: string,
    body?: any,
    withAuth = false
  ): Promise<T> {
    const headers: Record<string, string> = { 'Content-Type': 'application/json' }
    if (withAuth && this.accessToken) {
      headers['Authorization'] = `Bearer ${this.accessToken}`
    }

    const res = await fetch(`${this.baseUrl}${path}`, {
      method,
      headers,
      body: body ? JSON.stringify(body) : undefined,
    })

    // Auto-refresh on 401
    if (res.status === 401 && withAuth && this.refreshToken) {
      await this.refresh()
      return this.request(path, method, body, withAuth)
    }

    if (!res.ok) {
      const error = await res.json()
      throw new Error(error.message ?? 'Request failed')
    }

    return res.json()
  }
}
```

---

## 12. Admin Dashboard

A minimal React app (Vite) served as static files from the auth server at `/admin`.

### Pages
1. **Users** — searchable table, show email / role / created date / status. Can deactivate users.
2. **Sessions** — list active refresh tokens by user. Button to revoke all sessions for a user.
3. **Login Attempts** — table of recent attempts, filterable by success/fail and IP.
4. **Stats** — total users, active sessions, failed attempts in last 24h.

### Admin Authentication
Admin routes use a separate long-lived token issued via:
```bash
curl -X POST localhost:4000/admin/token \
  -H "Content-Type: application/json" \
  -d '{ "secret": "your-admin-secret-from-env" }'
```

---

## 13. Testing

### Unit Tests (Vitest)
Test service logic in isolation — mock the DB and cache adapters.

```typescript
// tests/unit/auth.service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { loginUser } from '../../src/services/auth.service.js'

const mockDb = {
  queryOne: vi.fn(),
  query: vi.fn(),
}
const mockCache = {
  get: vi.fn().mockResolvedValue(null),
  set: vi.fn(),
  increment: vi.fn().mockResolvedValue(1),
  expire: vi.fn(),
  del: vi.fn(),
}

describe('loginUser', () => {
  beforeEach(() => vi.clearAllMocks())

  it('throws UNAUTHORIZED for non-existent user', async () => {
    mockDb.queryOne.mockResolvedValue(null)
    await expect(loginUser('bad@test.com', 'pass', '127.0.0.1', mockDb as any, mockCache as any))
      .rejects.toMatchObject({ code: 'UNAUTHORIZED' })
  })

  it('returns tokens on valid credentials', async () => {
    // ... test with valid hash
  })

  it('locks account after 5 failed attempts', async () => {
    // ... test lockout threshold
  })
})
```

### Integration Tests
Test real HTTP endpoints against a test database.

```typescript
// tests/integration/auth.routes.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest'
import supertest from 'supertest'
import { buildServer } from '../../src/server.js'

let app: any

beforeAll(async () => {
  app = await buildServer({ env: 'test' })
  await app.ready()
})
afterAll(() => app.close())

describe('POST /auth/register', () => {
  it('registers a new user', async () => {
    const res = await supertest(app.server)
      .post('/auth/register')
      .send({ email: 'test@example.com', password: 'Str0ngP@ss!' })
    expect(res.status).toBe(201)
    expect(res.body.user.email).toBe('test@example.com')
  })

  it('rejects duplicate email with 409', async () => {
    await supertest(app.server).post('/auth/register')
      .send({ email: 'dupe@example.com', password: 'Str0ngP@ss!' })
    const res = await supertest(app.server).post('/auth/register')
      .send({ email: 'dupe@example.com', password: 'Str0ngP@ss!' })
    expect(res.status).toBe(409)
  })
})
```

### Test Coverage Targets
| Area | Target |
|---|---|
| Service layer | 90%+ |
| Route handlers | 80%+ |
| Utility functions | 100% |
| Adapter interfaces | 70%+ |

---

## 14. Error Handling Standard

All errors follow this shape:
```json
{
  "error": "ERROR_CODE",
  "message": "Human readable message",
  "details": {}    // optional, omitted in production for security-sensitive errors
}
```

### Error Codes
| Code | HTTP Status | Meaning |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Request body failed schema validation |
| `UNAUTHORIZED` | 401 | Invalid credentials or missing token |
| `TOKEN_EXPIRED` | 401 | JWT has expired |
| `INVALID_TOKEN` | 401 | JWT is malformed or signature invalid |
| `TOKEN_REUSE_DETECTED` | 401 | Refresh token reuse — all sessions revoked |
| `FORBIDDEN` | 403 | Valid token but insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `CONFLICT` | 409 | Resource already exists (duplicate email) |
| `TOO_MANY_REQUESTS` | 429 | Rate limit exceeded |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

### Custom Error Class (`src/utils/errors.ts`)
```typescript
export class AppError extends Error {
  constructor(
    public code: string,
    message: string,
    public statusCode: number = 400
  ) {
    super(message)
    this.name = 'AppError'
  }
}
```

---

## 15. Deployment

### Docker Compose (local development + self-hosting)
```yaml
# docker-compose.yml
version: '3.9'

services:
  authcore:
    build: .
    ports:
      - "4000:4000"
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://authcore:password@postgres:5432/authcore
      REDIS_URL: redis://redis:6379
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: authcore
      POSTGRES_USER: authcore
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U authcore"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### Quickstart (for consuming developers)
```bash
# 1. Clone and configure
git clone https://github.com/you/authcore.git
cd authcore
cp .env.example .env
# Edit .env with your secrets

# 2. Start everything
docker-compose up -d

# 3. Verify it's running
curl http://localhost:4000/health
# { "status": "ok", "version": "1.0.0" }
```

---

## 16. Open Source Checklist

Before making the repo public, verify:

### Repository
- [ ] `README.md` — overview, quickstart, configuration reference, API summary
- [ ] `CONTRIBUTING.md` — how to run locally, code style, PR process
- [ ] `CHANGELOG.md` — version history
- [ ] `LICENSE` — MIT recommended
- [ ] `.gitignore` — `.env`, `node_modules`, build output, logs
- [ ] No secrets, credentials, or personal data in Git history

### Code Quality
- [ ] All public functions have JSDoc comments
- [ ] No `TODO` or `FIXME` comments in main branch
- [ ] ESLint passes with zero errors
- [ ] All tests pass
- [ ] Test coverage meets targets (see Section 13)

### Security
- [ ] OWASP ZAP scan completed — findings documented
- [ ] `npm audit` passes with no high/critical
- [ ] Threat model document written and included in `/docs`
- [ ] Security policy (`SECURITY.md`) explaining how to report vulnerabilities

### Developer Experience
- [ ] Quickstart works on a fresh machine in under 5 minutes
- [ ] Swagger/OpenAPI docs accessible at `/docs`
- [ ] SDK published to npm (or clearly documented how to use it)
- [ ] At least one example app in `/examples` directory

---

*AuthCore — built with security-first defaults, designed to be dropped into any project.*
