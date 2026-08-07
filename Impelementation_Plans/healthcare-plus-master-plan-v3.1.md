# healthcare+ — Master Architecture Specification & Implementation Plan (v3.1 Enhanced)

> **Document Version**: v3.1 — Enhanced & Bug-Patched (Production Build-Ready).
> **Supersedes**: v3.0 Final. This version merges the detailed core-workflow document into the master spec, closes every open item flagged in the v2.0→v3.0 review cycle, and adds the missing tables/functions that were referenced but never defined.
> **Target Stack**: PERN — PostgreSQL 16+ (PostGIS), Express.js / Node.js 20+ LTS, React 18+ Vite.
> **Compliance**: India DPDP Act 2023, TRAI SMS Regulations, ABDM ABHA Standard.

---

## 0. Change Log — v3.0 → v3.1

| # | Area | v3.0 Gap | v3.1 Fix |
|---|---|---|---|
| 1 | RLS coverage | Only `appointments` and `billing_receipts` protected | RLS + policy added to **every** tenant-scoped table (11 tables) |
| 2 | `allocateTokenFallback` | Referenced, never defined | Fully specified with correct Postgres `FOR UPDATE` syntax |
| 3 | `authorizeRoomAccess` | Referenced, never defined | Fully specified per room type (queue / emergency) |
| 4 | Lite Appointment tokens | Fractional token (`15.5`) generated with no concurrency control — two simultaneous lite bookings could collide | Redis-based atomic lite-slot lock added |
| 5 | `mfa_secret` storage | Plaintext column | AES-256-GCM application-layer encryption specified |
| 6 | Doctor–tenant booking integrity | Nothing stops booking a doctor at an unaffiliated hospital | DB trigger + service-layer check added |
| 7 | Idempotency key | Column existed, never checked in code | Idempotency middleware specified |
| 8 | Departments | Referenced in admin workflows, no table existed | `departments` table added, FK'd from `doctor_affiliations` |
| 9 | Hospital branding/settings | Mentioned in vision, dropped from schema | `tenant_settings` table added |
| 10 | In-app notifications | BullMQ dispatches external channels only, no persistence | `notifications` table added for in-app bell/inbox |
| 11 | Core workflow detail | High-level only | Full state machines, edge cases, and sequence flows merged in from workflow doc |
| 12 | API surface | Partial endpoint list only | Full endpoint specification added (Part 4) |

Nothing here changes the product vision, removes a feature, or alters the phased roadmap — every change is a structural completion of something already promised elsewhere in the document.

---

# PART 1: BACKEND ARCHITECTURE

## 1.1 Infrastructure Topology (unchanged from v3.0)

```
                                  [ Cloudflare WAF / DNS ]
                                             |
                                  [ NGINX Reverse Proxy ]
                                             |
                  +--------------------------+--------------------------+
                  v                                                     v
      [ Express Node Cluster (PM2) ]                        [ React Frontend CDN ]
                  |
        +---------+---------+
        v                   v
[ PostgreSQL 16 Primary ]   [ Redis Cluster ]
(PostGIS + Read Replica)    (Cache + Pub/Sub + BullMQ + Token/Slot Locks)
        ^
   [ PgBouncer — Transaction Mode ]
```

---

## 1.2 Tenant Security Layer

### 1.2.1 Tenant Context Middleware (unchanged, already correct)

```javascript
export const enforceTenantContext = async (req, res, next) => {
  try {
    let tenantId = null;
    if (req.user?.role === 'super_admin') {
      const headerTenant = req.headers['x-tenant-id'];
      if (headerTenant) {
        const tenant = await tenantRepository.findActiveById(headerTenant);
        if (!tenant) return res.status(404).json({ error: 'Tenant invalid or deactivated' });
        tenantId = tenant.id;
      }
    } else if (req.user) {
      tenantId = req.user.tenantId;
    }
    if (!tenantId && req.isTenantRoute) {
      return res.status(403).json({ error: 'Access Denied: Tenant context required.' });
    }
    req.tenantId = tenantId;
    next();
  } catch (err) { next(err); }
};
```

### 1.2.2 PgBouncer-Safe RLS Wrapper (unchanged, already correct)

```javascript
export async function withTenantContext(tenantId, fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query('SELECT set_config($1, $2, true)', ['app.current_tenant_id', tenantId]);
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}
```

### 1.2.3 Idempotency Middleware (new — closes v3.0 gap #7)

```javascript
// middleware/idempotency.middleware.js
export const idempotencyGuard = (resourceCheckFn) => async (req, res, next) => {
  const key = req.headers['idempotency-key'];
  if (!key) return res.status(400).json({ error: 'Idempotency-Key header required' });

  const existing = await resourceCheckFn(key); // e.g. appointmentRepository.findByIdempotencyKey(key)
  if (existing) {
    // Return the original result instead of creating a duplicate
    return res.status(200).json({ idempotent: true, data: existing });
  }
  req.idempotencyKey = key;
  next();
};

// Usage:
router.post('/appointments/book',
  authenticate, enforceTenantContext,
  idempotencyGuard((key) => appointmentRepository.findByIdempotencyKey(key)),
  appointmentController.bookAppointment
);
```

The booking service must persist `req.idempotencyKey` into `appointments.idempotency_key` in the same transaction that allocates the token and creates the pending billing receipt, so a retried request with the same key never double-books or double-charges.

---

## 1.3 Complete Database DDL (v3.1)

Deltas from v3.0 are marked `-- NEW` or `-- FIXED`. All v3.0 tables are retained.

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";

-- 1. TENANTS
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) UNIQUE NOT NULL,
    address TEXT NOT NULL,
    location GEOMETRY(Point, 4326),
    contact_email VARCHAR(255) NOT NULL,
    contact_phone VARCHAR(50) NOT NULL,
    signing_public_key TEXT NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_tenants_location ON tenants USING GIST(location);

-- 1a. TENANT SETTINGS (NEW — branding, hours, daily capacity for crowd status calc)
CREATE TABLE tenant_settings (
    tenant_id UUID PRIMARY KEY REFERENCES tenants(id) ON DELETE CASCADE,
    logo_url TEXT,
    primary_color VARCHAR(20),
    working_hours JSONB DEFAULT '{}'::jsonb, -- {"mon": ["09:00","18:00"], ...}
    daily_token_capacity_per_doctor INT DEFAULT 40, -- used by AI Crowd Status engine
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 2. USERS
CREATE TYPE user_role AS ENUM (
    'super_admin', 'hospital_admin', 'doctor', 'receptionist',
    'lab_tech', 'pharmacist', 'nurse', 'patient', 'ambulance_driver', 'support'
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE RESTRICT,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    role user_role NOT NULL,
    mfa_secret_encrypted TEXT,        -- FIXED: AES-256-GCM ciphertext, never plaintext
    mfa_secret_iv VARCHAR(32),        -- FIXED: initialization vector for decryption
    is_mfa_enabled BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_users_tenant_role ON users(tenant_id, role);

-- 3. DEPARTMENTS (NEW — was a free-text field on doctor_affiliations)
CREATE TABLE departments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, name)
);

-- 4. DOCTOR AFFILIATIONS
CREATE TABLE doctor_affiliations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    doctor_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    department_id UUID NOT NULL REFERENCES departments(id), -- FIXED: FK instead of free text
    consultation_fee NUMERIC(10,2) NOT NULL DEFAULT 500.00,
    lite_consultation_fee NUMERIC(10,2) NOT NULL DEFAULT 200.00, -- NEW: for Lite Appointments
    is_primary BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(doctor_id, tenant_id)
);

-- 5. HEALTHCARE PASSPORTS
CREATE TABLE patient_passports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    patient_id UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    abha_id VARCHAR(50) UNIQUE,
    blood_group VARCHAR(10),
    allergies JSONB DEFAULT '[]'::jsonb,
    chronic_conditions JSONB DEFAULT '[]'::jsonb,
    emergency_contact JSONB NOT NULL,
    is_anonymized BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 6. CONSENT GRANTS
CREATE TABLE passport_consent_grants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    passport_id UUID NOT NULL REFERENCES patient_passports(id) ON DELETE CASCADE,
    grantee_doctor_id UUID NOT NULL REFERENCES users(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),  -- FIXED: needed for RLS scoping
    grant_type VARCHAR(50) NOT NULL,
    granted_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ
);
CREATE INDEX idx_consent_active ON passport_consent_grants(passport_id, grantee_doctor_id) WHERE revoked_at IS NULL;

-- 7. APPOINTMENTS
CREATE TYPE appointment_type AS ENUM ('REGULAR', 'LITE_FOLLOWUP');
CREATE TYPE appointment_status AS ENUM ('PENDING_PAYMENT', 'CONFIRMED', 'IN_PROGRESS', 'COMPLETED', 'NO_SHOW', 'CANCELLED');

CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE RESTRICT,
    patient_id UUID NOT NULL REFERENCES users(id),
    doctor_id UUID NOT NULL REFERENCES users(id),
    appointment_date DATE NOT NULL,
    slot_time TIME NOT NULL,
    token_number NUMERIC(6,1) NOT NULL,
    type appointment_type DEFAULT 'REGULAR',
    status appointment_status DEFAULT 'PENDING_PAYMENT',
    consultation_fee NUMERIC(10,2) NOT NULL,
    idempotency_key VARCHAR(255) UNIQUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, doctor_id, appointment_date, token_number)
);
CREATE INDEX idx_appointments_queue ON appointments(tenant_id, doctor_id, appointment_date, status);

-- FIXED: DB-level trigger guaranteeing doctor is actually affiliated with the booking tenant
CREATE OR REPLACE FUNCTION check_doctor_affiliation() RETURNS TRIGGER AS $$
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM doctor_affiliations
    WHERE doctor_id = NEW.doctor_id AND tenant_id = NEW.tenant_id AND is_active = TRUE
  ) THEN
    RAISE EXCEPTION 'Doctor % is not an active affiliate of tenant %', NEW.doctor_id, NEW.tenant_id;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_check_doctor_affiliation
  BEFORE INSERT ON appointments
  FOR EACH ROW EXECUTE FUNCTION check_doctor_affiliation();

-- 8. PRESCRIPTIONS
CREATE TABLE prescriptions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    appointment_id UUID REFERENCES appointments(id),
    patient_id UUID NOT NULL REFERENCES users(id),
    doctor_id UUID NOT NULL REFERENCES users(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    diagnosis TEXT NOT NULL,
    instructions TEXT,
    cds_evaluation JSONB, -- NEW: stores the Hybrid CDS Engine result for audit trail
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE prescription_medicines (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    prescription_id UUID NOT NULL REFERENCES prescriptions(id) ON DELETE CASCADE,
    medicine_name VARCHAR(255) NOT NULL,
    dosage_per_day VARCHAR(50) NOT NULL,
    duration_days INT NOT NULL,
    special_instructions VARCHAR(255)
);

-- 9. PHARMACY ORDERS
CREATE TYPE pharmacy_order_status AS ENUM ('PENDING_CONFIRMATION', 'RECEIVED', 'PACKED', 'PAID', 'COMPLETED', 'CANCELLED');

CREATE TABLE pharmacy_orders (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    patient_id UUID NOT NULL REFERENCES users(id),
    prescription_id UUID REFERENCES prescriptions(id),
    status pharmacy_order_status DEFAULT 'PENDING_CONFIRMATION',
    total_amount NUMERIC(10,2) NOT NULL,
    is_paid BOOLEAN DEFAULT FALSE,
    reminders_activated BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 10. LAB REQUESTS
CREATE TYPE lab_status AS ENUM ('REQUESTED', 'PAID', 'SAMPLE_COLLECTED', 'PROCESSING', 'COMPLETED');

CREATE TABLE lab_requests (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    patient_id UUID NOT NULL REFERENCES users(id),
    doctor_id UUID REFERENCES users(id),
    test_name VARCHAR(255) NOT NULL,
    cost NUMERIC(10,2) NOT NULL,
    status lab_status DEFAULT 'REQUESTED',
    report_file_url TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 11. BILLING
CREATE TABLE billing_receipts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    patient_id UUID NOT NULL REFERENCES users(id),
    appointment_id UUID REFERENCES appointments(id) ON DELETE SET NULL,
    lab_request_id UUID REFERENCES lab_requests(id) ON DELETE SET NULL,
    pharmacy_order_id UUID REFERENCES pharmacy_orders(id) ON DELETE SET NULL,
    receipt_number VARCHAR(100) UNIQUE NOT NULL,
    total_amount NUMERIC(10,2) NOT NULL,
    tax_amount NUMERIC(10,2) DEFAULT 0.00,
    discount_amount NUMERIC(10,2) DEFAULT 0.00,
    is_paid BOOLEAN DEFAULT FALSE,
    payment_method VARCHAR(50),
    transaction_ref VARCHAR(255),
    digital_signature TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_billing_has_source CHECK (
      appointment_id IS NOT NULL OR lab_request_id IS NOT NULL OR pharmacy_order_id IS NOT NULL
    )
);

CREATE TABLE receipt_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    receipt_id UUID NOT NULL REFERENCES billing_receipts(id) ON DELETE CASCADE,
    item_type VARCHAR(50) NOT NULL,
    description VARCHAR(255) NOT NULL,
    quantity INT NOT NULL DEFAULT 1,
    unit_price NUMERIC(10,2) NOT NULL,
    total_price NUMERIC(10,2) NOT NULL
);

-- 12. AMBULANCE FLEET & EMERGENCY
CREATE TABLE ambulance_units (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id),
    driver_id UUID NOT NULL REFERENCES users(id),
    vehicle_number VARCHAR(50) NOT NULL,
    current_location GEOMETRY(Point, 4326),
    is_available BOOLEAN DEFAULT TRUE,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_ambulance_location ON ambulance_units USING GIST(current_location);

CREATE TYPE emergency_status AS ENUM (
    'BROADCASTING', 'ACCEPTED', 'EN_ROUTE_PATIENT', 'PATIENT_PICKED', 'ARRIVED_HOSPITAL', 'ESCALATED_108', 'CANCELLED'
);

CREATE TABLE emergency_dispatches (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    patient_id UUID NOT NULL REFERENCES users(id),
    assigned_tenant_id UUID REFERENCES tenants(id),
    assigned_driver_id UUID REFERENCES users(id),
    patient_location GEOMETRY(Point, 4326) NOT NULL,
    status emergency_status DEFAULT 'BROADCASTING',
    eta_minutes INT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_emergency_location ON emergency_dispatches USING GIST(patient_location);

-- 13. AUDIT & CONSENT LEDGER
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    action VARCHAR(100) NOT NULL,
    actor_user_id UUID REFERENCES users(id),
    target_passport_id UUID REFERENCES patient_passports(id),
    metadata JSONB DEFAULT '{}'::jsonb,
    ip_address INET,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
-- Append-only: application DB role has INSERT/SELECT only, no UPDATE/DELETE grant

CREATE TABLE notification_consents (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id),
    channel VARCHAR(20) NOT NULL,
    consented BOOLEAN NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 14. IN-APP NOTIFICATIONS (NEW — persisted inbox/bell feed, distinct from SMS/Email dispatch)
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id),
    tenant_id UUID REFERENCES tenants(id),
    category VARCHAR(50) NOT NULL, -- 'APPOINTMENT' | 'LAB' | 'PHARMACY' | 'BILLING' | 'QUEUE' | 'EMERGENCY'
    title VARCHAR(255) NOT NULL,
    body TEXT,
    is_read BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_notifications_unread ON notifications(user_id, is_read) WHERE is_read = FALSE;

-- ============================================================
-- ROW LEVEL SECURITY — FIXED: applied to every tenant-scoped table
-- ============================================================
DO $$
DECLARE
  t TEXT;
BEGIN
  FOREACH t IN ARRAY ARRAY[
    'users', 'departments', 'doctor_affiliations', 'appointments',
    'prescriptions', 'pharmacy_orders', 'lab_requests', 'billing_receipts',
    'ambulance_units', 'emergency_dispatches', 'passport_consent_grants'
  ]
  LOOP
    EXECUTE format('ALTER TABLE %I ENABLE ROW LEVEL SECURITY;', t);
    EXECUTE format(
      'CREATE POLICY tenant_isolation_%1$I ON %1$I
         FOR ALL USING (
           tenant_id = NULLIF(current_setting(''app.current_tenant_id'', true), '''')::uuid
           OR current_setting(''app.current_tenant_id'', true) IS NULL -- super_admin bypass handled at app layer
         );', t
    );
  END LOOP;
END $$;

-- emergency_dispatches and ambulance_units use assigned_tenant_id / tenant_id respectively;
-- confirm column name matches during migration — assigned_tenant_id is nullable pre-acceptance,
-- so its policy additionally allows rows where assigned_tenant_id IS NULL (still broadcasting, unassigned).
DROP POLICY tenant_isolation_emergency_dispatches ON emergency_dispatches;
CREATE POLICY tenant_isolation_emergency_dispatches ON emergency_dispatches
    FOR ALL USING (
      assigned_tenant_id = NULLIF(current_setting('app.current_tenant_id', true), '')::uuid
      OR assigned_tenant_id IS NULL
    );
```

> **Migration note**: `users` now has RLS enabled. Auth routes (`/login`, `/register`) run **before** tenant context exists, so the auth service must connect using a Postgres role that bypasses RLS (`BYPASSRLS`) strictly for the login/lookup-by-email path, or perform that lookup via a `SECURITY DEFINER` function. This must be documented clearly for whoever implements Phase B1, since it's the one legitimate RLS exception in the schema.

---

## 1.4 Previously-Stubbed Functions — Now Fully Specified

### 1.4.1 `allocateTokenFallback` (closes v3.0 gap #2)

```javascript
// services/queue.service.js
async function allocateTokenFallback(tenantId, doctorId, date) {
  return withTenantContext(tenantId, async (client) => {
    const { rows } = await client.query(
      `SELECT COALESCE(MAX(token_number), 0) + 1 AS next_token
       FROM appointments
       WHERE tenant_id = $1 AND doctor_id = $2 AND appointment_date = $3
         AND type = 'REGULAR'
       FOR UPDATE`,
      [tenantId, doctorId, date]
    );
    return rows[0].next_token;
  });
}
```
Postgres `FOR UPDATE` syntax only — no `WITH (UPDLOCK, HOLDLOCK)` (that was a SQL Server construct mistakenly used in an earlier draft and must not reappear in the implementation).

### 1.4.2 `authorizeRoomAccess` (closes v3.0 gap #3)

```javascript
// services/socket.authorization.js
export async function authorizeRoomAccess(user, room) {
  const [type, ...rest] = room.split(':');

  if (type === 'queue') {
    const [tenantId, doctorId] = rest;
    if (user.role === 'patient') {
      // Patient may only join if they hold an active appointment with this doctor today
      return appointmentRepository.hasActiveAppointment(user.id, doctorId, tenantId);
    }
    // Staff may join only their own tenant's queue room
    return user.tenantId === tenantId;
  }

  if (type === 'emergency') {
    const [dispatchId] = rest;
    const dispatch = await emergencyRepository.findById(dispatchId);
    if (!dispatch) return false;
    // Only the requesting patient, the assigned driver, or ER staff of the assigned hospital may join
    return (
      dispatch.patient_id === user.id ||
      dispatch.assigned_driver_id === user.id ||
      (user.tenantId && user.tenantId === dispatch.assigned_tenant_id)
    );
  }

  return false; // deny by default for any unrecognized room type
}
```

### 1.4.3 Lite Appointment Atomic Slot Lock (closes v3.0 gap #4)

The original `allocateLiteToken` simply computed `currentActiveToken + 0.5` with no concurrency guard — two receptionists inserting a lite slot around the same active token at the same moment could both compute `15.5` and collide on the `UNIQUE(tenant_id, doctor_id, appointment_date, token_number)` constraint (which is a safe failure, but with no retry logic the second request just errors out). Fixed version adds a short Redis lock plus a retry-with-next-half-slot:

```javascript
static async allocateLiteToken(tenantId, doctorId, date) {
  const lockKey = `lite_lock:${tenantId}:${doctorId}:${date}`;
  const lock = await redis.set(lockKey, '1', 'NX', 'EX', 5); // 5s exclusive lock
  if (!lock) throw new Error('LITE_SLOT_BUSY: try again in a moment');

  try {
    return await withTenantContext(tenantId, async (client) => {
      const { rows } = await client.query(
        `SELECT token_number FROM appointments
         WHERE tenant_id = $1 AND doctor_id = $2 AND appointment_date = $3
           AND status = 'IN_PROGRESS'
         ORDER BY token_number DESC LIMIT 1 FOR UPDATE`,
        [tenantId, doctorId, date]
      );
      const active = rows[0]?.token_number ?? 0;
      let candidate = parseFloat(active) + 0.5;

      // If that half-slot is already taken (rare, e.g. two lite bookings before the next
      // regular token advances), step forward in 0.1 increments until a free slot is found.
      while (await slotTaken(client, tenantId, doctorId, date, candidate)) {
        candidate = Math.round((candidate + 0.1) * 10) / 10;
      }
      return candidate;
    });
  } finally {
    await redis.del(lockKey);
  }
}
```

### 1.4.4 MFA Secret Encryption (closes v3.0 gap #5)

```javascript
// utils/mfaCrypto.js
import crypto from 'crypto';
const ALGO = 'aes-256-gcm';
const KEY = Buffer.from(process.env.MFA_ENCRYPTION_KEY, 'hex'); // 32-byte key, from Secrets Manager

export function encryptMfaSecret(plainSecret) {
  const iv = crypto.randomBytes(12);
  const cipher = crypto.createCipheriv(ALGO, KEY, iv);
  const encrypted = Buffer.concat([cipher.update(plainSecret, 'utf8'), cipher.final()]);
  const authTag = cipher.getAuthTag();
  return {
    ciphertext: Buffer.concat([encrypted, authTag]).toString('base64'),
    iv: iv.toString('base64'),
  };
}

export function decryptMfaSecret(ciphertextB64, ivB64) {
  const data = Buffer.from(ciphertextB64, 'base64');
  const iv = Buffer.from(ivB64, 'base64');
  const authTag = data.subarray(data.length - 16);
  const encrypted = data.subarray(0, data.length - 16);
  const decipher = crypto.createDecipheriv(ALGO, KEY, iv);
  decipher.setAuthTag(authTag);
  return Buffer.concat([decipher.update(encrypted), decipher.final()]).toString('utf8');
}
```
Stored in `users.mfa_secret_encrypted` / `users.mfa_secret_iv`, decrypted only in-memory at TOTP verification time, never logged.

---

## 1.5 AI Crowd Status Engine (detailed)

Uses `tenant_settings.daily_token_capacity_per_doctor` rather than a hardcoded constant, aggregated across all active doctors at a tenant:

```javascript
export const computeCrowdStatus = (activeTokensToday, dailyCapacity) => {
  const ratio = activeTokensToday / dailyCapacity;
  if (ratio < 0.3) return { label: 'Low', code: 'LOW', color: 'GREEN' };
  if (ratio < 0.7) return { label: 'Moderate', code: 'MODERATE', color: 'YELLOW' };
  return { label: 'High', code: 'HIGH', color: 'RED' };
};

// Aggregation query (cached in Redis with 60s TTL, since this backs a
// hospital-discovery list that doesn't need per-second freshness):
async function getTenantCrowdStatus(tenantId, date) {
  const cacheKey = `crowd:${tenantId}:${date}`;
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const { activeTokens, capacity } = await queueRepository.getDailyLoad(tenantId, date);
  const result = computeCrowdStatus(activeTokens, capacity);
  await redis.set(cacheKey, JSON.stringify(result), 'EX', 60);
  return result;
}
```

---

# PART 2: CORE WORKFLOWS (Merged & Expanded)

## 2.1 User Home Dashboard

**AI Health Assistant** (top of screen): free-text symptom box → `POST /triage/analyze` → Gemini structured-JSON response mapping to a department name that must exist in that hospital's `departments` table for the suggestion to be actionable — if the AI suggests a department a given nearby hospital doesn't have, the hospital card simply won't surface a "Book" shortcut for that specialty, avoiding dead-end suggestions. Every triage response is rendered with the fixed safety disclaimer beneath it and never blocks the visible SOS button.

**Nearby Hospitals card grid**: nearest 4 by `ST_Distance`, each showing name, distance, rating, specialties (joined from `departments`), estimated consultation fee (from `doctor_affiliations`), and AI Crowd Status badge (§1.5). Cards refresh crowd status via a lightweight polling fetch every 60s — not a WebSocket subscription, since crowd status is inherently low-frequency data and doesn't warrant a persistent connection per browsing patient.

**Emergency SOS** — full state machine:

```
IDLE
  → (hold 3s) → CONFIRM_MODAL
      → (confirm) → LOCATING            [HTML5 Geolocation]
          → (coords acquired) → BROADCASTING   [PostGIS 10km search]
              → (0 candidates) → ESCALATED_108   [terminal state, shown to patient]
              → (≥1 candidate) → candidates notified via socket
                  → (driver accepts, atomic lock wins) → ACCEPTED
                      → EN_ROUTE_PATIENT → PATIENT_PICKED → ARRIVED_HOSPITAL  [terminal]
                  → (15s no acceptance) → ESCALATED_108   [terminal]
      → (cancel before driver accepts) → CANCELLED   [terminal, requires reason]
```
The patient-facing screen literally renders these states as: *"Searching for nearby ambulances…" → "Driver Accepted! Ramesh Kumar (KA-01-EA-1234)" → live map with dynamic ETA → "Destination Hospital ER Pre-Notified."* Cancellation after a driver has been assigned (`EN_ROUTE_PATIENT` or later) is intentionally **not** exposed in the UI — only pre-acceptance cancellation is allowed, to avoid drivers being dispatched and then silently abandoned mid-route without hospital/admin visibility.

## 2.2 Patient Dashboard

Aggregates, via React Query, four independently-cached server slices: upcoming appointments, live queue ticket (subscribed over WebSocket only while an appointment is `CONFIRMED`/`IN_PROGRESS` today — not persistently), active prescriptions with dose-taken checkboxes (writes to a `dose_logs` companion table — see §2.6), lab reports, billing history, and the Healthcare Passport timeline. Clicking any appointment routes into that hospital's isolated workspace context (`enforceTenantContext` picks up the tenant from the appointment's `tenant_id`, not a manually re-typed header).

## 2.3 Hospital Workspace & Booking

`Department → Doctor → Slot → Payment → Confirmed`. On the slot-selection screen, a **"Lite Appointment (Report Review / Prescription Update)"** toggle is only shown if the patient already has a `COMPLETED` appointment with that doctor within the last 90 days — Lite Appointments are explicitly a follow-up mechanism, not a way to skip the regular queue for a first visit. Selecting it swaps the fee to `doctor_affiliations.lite_consultation_fee` and calls `allocateLiteToken` (§1.4.3) instead of the regular Redis `INCR` counter.

**Live Queue Ticker** — WebSocket room `queue:{tenantId}:{doctorId}`, joined only after `authorizeRoomAccess` (§1.4.2) confirms the patient holds a same-day active appointment. Payload on `queue:updated`:
```json
{ "currentToken": 14, "yourToken": 18, "patientsAhead": 3, "estimatedWaitMinutes": 15 }
```

## 2.4 Doctor Desk

Patient queue list with **Call Next**, **Start Consultation**, **Mark No-Show** (writes `NO_SHOW` status — distinct from `CANCELLED`, enabling future no-show-rate analytics per doctor). Passport viewer surfaces allergy/chronic-condition banners at the top, sourced from `patient_passports`, gated by an active `passport_consent_grants` row (auto-granted for 24h on booking, per §Phase B2).

**e-Prescription Generator** calls `HybridCDSEngine.evaluatePrescription` (§Phase B5) before allowing signature. A `REJECTED` result renders as a blocking red banner the doctor cannot dismiss without editing the prescription; an `APPROVED` result shows the AI-generated plain-language summary as a dismissible, clearly-labeled "AI explanation, not a safety check" note — so the UI itself reinforces which layer is authoritative.

## 2.5 Pharmacy Workflow (full state machine)

```
Doctor signs prescription
   → Patient confirms purchase from hospital pharmacy [pharmacy_orders: PENDING_CONFIRMATION]
       → Pharmacy staff marks RECEIVED
           → Pharmacy staff marks PACKED  → notification sent to patient
               → Patient pays (if not pre-paid) → PAID
                   → Patient collects → COMPLETED
                       → reminders_activated = TRUE
                       → dose_logs schedule generated from prescription_medicines
```
If a patient never confirms purchase within 48 hours of prescription signing, the order auto-expires to `CANCELLED` via a BullMQ delayed job — this prevents an indefinite `PENDING_CONFIRMATION` backlog cluttering the pharmacy console.

## 2.6 Medicine Reminders — `dose_logs` (new supporting table)

Referenced by the Patient Dashboard's dose-taken checkboxes but never defined in earlier drafts:

```sql
CREATE TABLE dose_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    prescription_medicine_id UUID NOT NULL REFERENCES prescription_medicines(id) ON DELETE CASCADE,
    patient_id UUID NOT NULL REFERENCES users(id),
    scheduled_for TIMESTAMPTZ NOT NULL,
    taken_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_dose_logs_pending ON dose_logs(patient_id, scheduled_for) WHERE taken_at IS NULL;
```
BullMQ generates `dose_logs` rows on pharmacy order `COMPLETED`, based on `dosage_per_day` (e.g. `"1-0-1"` → two rows/day for `duration_days`), and schedules a push notification per row.

## 2.7 Laboratory Workflow (full state machine)

```
Doctor creates request [lab_requests: REQUESTED]
   → Patient views cost, confirms & pays → PAID
       → Lab marks SAMPLE_COLLECTED
           → Lab marks PROCESSING
               → Lab uploads PDF (ClamAV scan + magic-byte check) → COMPLETED
                   → report_file_url attached
                   → in-app notification (notifications table) fired to both doctor and patient
                   → visible immediately in Healthcare Passport timeline
```

## 2.8 Hospital Administration

Scoped entirely by `enforceTenantContext` + RLS — a `hospital_admin`'s JWT `tenantId` claim is the sole source of tenant scope for every admin query, so cross-hospital data simply cannot appear regardless of what the admin UI requests. Admin panel manages: `departments`, `doctor_affiliations` (including fee structures), `tenant_settings` (branding/hours/capacity), live queue monitoring with manual token override (writes an `audit_logs` row per override — token overrides are exactly the kind of action that needs an accountable trail), lab/pharmacy pricing, ER console, and revenue analytics computed from `billing_receipts` scoped to their tenant.

---

# PART 3: FRONTEND (retained from v3.0, unchanged)

Design token system, primitive component suite, dashboard layouts, and console specifications from v3.0 Part 2 (Phases F1–F6) remain valid as written and are not reproduced here to avoid duplication — see v3.0 §F1–F6.

---

# PART 4: API SURFACE (new — closes v3.0 gap #12)

| Method | Endpoint | Auth | Notes |
|---|---|---|---|
| `POST` | `/api/v1/auth/register` | Public | Runs pre-tenant-context; uses `BYPASSRLS` lookup path |
| `POST` | `/api/v1/auth/login` | Public | Issues access + refresh token pair |
| `POST` | `/api/v1/auth/refresh` | Refresh token | Rotates token; reuse triggers family revocation |
| `POST` | `/api/v1/auth/mfa/verify` | Session | Decrypts `mfa_secret_encrypted` in-memory only |
| `GET` | `/api/v1/hospitals/nearby` | Auth | PostGIS distance query + cached crowd status |
| `POST` | `/api/v1/triage/analyze` | Auth | SSE-streamed, department-only output |
| `POST` | `/api/v1/appointments/book` | Patient | Requires `Idempotency-Key` header |
| `GET` | `/api/v1/queue/live/:doctorId` | Auth | REST snapshot; live updates via socket room |
| `POST` | `/api/v1/emergency/trigger` | Patient | Zero-candidate path auto-escalates |
| `POST` | `/api/v1/emergency/accept/:id` | Driver | Atomic conditional update |
| `GET` | `/api/v1/passport/timeline` | Auth + Consent | Requires active `passport_consent_grants` row |
| `POST` | `/api/v1/lab/reports/upload` | Lab Tech | ClamAV-scanned multipart upload |
| `POST` | `/api/v1/pharmacy/orders/:id/confirm` | Patient | Starts 48h auto-expiry timer |
| `PATCH` | `/api/v1/dose-logs/:id/taken` | Patient | Marks a reminder as taken |
| `GET` | `/api/v1/admin/audit-logs` | Hospital Admin | Tenant-scoped via RLS, append-only source |

---

# PART 5: EDGE CASES & FAILURE MODES (new)

| Scenario | Handling |
|---|---|
| Redis cluster unavailable during booking | Falls to `allocateTokenFallback` (§1.4.1); if Postgres is also degraded, booking endpoint returns 503 rather than issuing an unprotected token |
| Doctor deactivated mid-day with pending queue | Existing `CONFIRMED` tokens remain visible with a "doctor unavailable — contact reception" banner; no auto-cancellation, since only a human can safely triage that queue |
| Two lite-appointment requests within the same 5s Redis lock window | Second request receives `LITE_SLOT_BUSY`, frontend retries automatically once after a short delay |
| Patient revokes doctor consent mid-appointment | Consent revocation takes effect on next request, not retroactively — doctor's already-open passport view for the in-progress consultation remains visible until the appointment reaches `COMPLETED`, avoiding an active consult being cut off mid-visit |
| Ambulance driver goes offline after accepting | No automatic re-broadcast is built into the state machine (deliberately, to avoid patient confusion from repeated reassignment); this is flagged to ER staff via the `emergency:{dispatchId}` room for manual escalation |
| Lab PDF fails ClamAV scan | Upload rejected before storage, lab tech sees an explicit "file failed security scan" error, `lab_requests.status` remains `PROCESSING` |

---

## Final Production Readiness Checklist (v3.1)

All items from v3.0 §Final Checklist, plus:
- [ ] RLS enabled and policy-tested on all 11 tenant-scoped tables (not just 2)
- [ ] `users` table RLS bypass path for pre-auth login verified not to leak cross-tenant emails
- [ ] `allocateTokenFallback` and `authorizeRoomAccess` unit-tested directly (not just referenced)
- [ ] Lite Appointment concurrent-booking test: 2 simultaneous lite requests for the same doctor/date resolve to distinct token numbers
- [ ] `mfa_secret_encrypted` verified never appears in logs, error messages, or API responses
- [ ] Doctor-affiliation trigger verified to reject booking a doctor at a non-affiliated tenant
- [ ] Idempotency-Key replay test: identical retried booking request returns the original appointment, not a duplicate
- [ ] Pharmacy order 48h auto-expiry BullMQ job verified
- [ ] `dose_logs` generation verified against `dosage_per_day` parsing for at least 3 real-world dosage string formats (e.g. `"1-0-1"`, `"1-1-1-1"`, `"SOS"`)

---

*End of v3.1. Where this document conflicts with v3.0, v3.1 governs. Where v3.1 is silent, v3.0 Part 2 (Frontend) and Part 3 (Roadmap) remain in force.*
