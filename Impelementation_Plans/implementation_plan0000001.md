# healthcare+ — Ultimate Master Architecture & Implementation Blueprint (v4.0 Final)

> **Document Status**: APPROVED MASTER BLUEPRINT (Production Build-Ready).
> **Target Tech Stack**: PERN — PostgreSQL 16+ (PostGIS), Express.js / Node.js 20+ LTS, React 18+ Vite.
> **Architecture Model**: Multi-Hospital SaaS Healthcare Network Platform & Universal Patient Healthcare Passport.
> **Primary Compliance Target**: India Digital Personal Data Protection (DPDP) Act 2023, TRAI SMS Regulations, ABDM ABHA Standard.

---

# SECTION 1: ARCHITECTURAL COMPARISON & JUSTIFICATION

## 1.1 Architectural Comparison Matrix

| Dimension | Legacy Django/SQLite Prototype | v1.0 Proposal Draft | v3.1 Draft Iteration | Ultimate PERN Blueprint (v4.0 Final) |
|:---|:---|:---|:---|:---|
| **Multi-Tenancy** | Single-hospital monolith. No tenant boundary. | Shared DB, header-trusted `x-tenant-id`. | Parameterized `set_config` RLS for 2 tables. | **Strict Row-Level Security (RLS) on all 12 operational tables**; header `x-tenant-id` untrusted; `withTenantContext` PgBouncer transaction wrapper. |
| **Database Engine** | SQLite (concurrency limit, no GIS). | PostgreSQL 16 + PostGIS. | PostgreSQL 16 + PostGIS (partial DDL). | **PostgreSQL 16 + PostGIS** with 15 complete DDL tables, GIST spatial indexes, DB triggers, and PgBouncer connection pooling. |
| **Token Queue Engine** | Sequential loop in memory. | Unspecified concurrency. | Basic Redis `INCR`. | **Atomic Redis `INCR` counter** with Postgres `SELECT FOR UPDATE` fallback + **Redis-locked Lite Token fractional allocation** (`15.5`). |
| **Emergency Ambulance Dispatch** | Static string location, FCFS endpoint. | PostGIS spatial search. | Atomic driver update (`RETURNING *`). | **PostGIS 10km spatial search**, atomic `UPDATE...WHERE status='BROADCASTING' RETURNING *` lock, and **immediate 0-driver 108/102 tele-dispatch fallback**. |
| **Clinical Decision Support (CDS)** | Simple string regex rules. | LLM replacing rules (safety risk). | Hybrid model proposed. | **Hybrid CDS Engine**: Deterministic core rule engine (blocking primary gate for pediatric/allergy/dosage checks) + LLM advisory summaries. |
| **Medical Identity & Compliance** | Single-tenant local patient rows. | Universal Healthcare Passport. | Basic consent grants. | **Universal Healthcare Passport** with ABHA ID linkage, patient consent grants (`APPOINTMENT_AUTO`, `EMERGENCY_OVERRIDE`, `MANUAL_GRANT`), and **DPDP Act 2023 soft anonymization**. |
| **Security & Authentication** | Plaintext passwords, mock JWTs. | Argon2id & RS256 JWTs. | MFA mentioned. | **Argon2id + RS256 JWT Rotation** with Redis token family revocation, **AES-256-GCM encrypted MFA secrets**, Ed25519 receipt signatures, and ClamAV PDF scanning. |
| **Real-Time Architecture** | Manual polling / page refresh. | Socket.io server. | Room structure outlined. | **Authenticated Socket.io** with JWT handshake verification & per-room authorization guard (`authorizeRoomAccess`). |
| **API Latency SLAs** | Unspecified. | Flat <100ms target. | Tiered SLA proposed. | **Tiered SLA Matrix**: Core REST <100ms, WebSockets <50ms, AI Symptom Triage <3s (SSE Streaming), File Uploads <5s. |

## 1.2 Technology Stack Justification

```
                                  [ Cloudflare WAF / DNS ]
                                             |
                                  [ NGINX Reverse Proxy ]
                                             |
                  +--------------------------+--------------------------+
                  |                                                     |
                  v                                                     v
      [ Express Node Cluster (PM2) ]                        [ React Frontend CDN ]
                  |                                         (Vite + TailwindCSS)
        +---------+---------+
        |                   |
        v                   v
[ PostgreSQL 16 Primary ]   [ Redis Cluster ]
(PostGIS + Read Replica)    (Cache + Pub/Sub + BullMQ + Token Locks)
        ^
   [ PgBouncer — Transaction Mode ]
```

1. **Database (PostgreSQL 16 + PostGIS)**: Selected over MySQL/MongoDB due to native spatial query support (`ST_DWithin`, `ST_Distance`), JSONB document storage for flexible health vitals, robust transactional integrity, and mature Row-Level Security (RLS) for multi-tenant SaaS isolation.
2. **Backend Framework (Express.js / Node.js 20+ LTS)**: Selected over Django/Python due to high non-blocking I/O throughput for real-time WebSocket connections (Socket.io), lightweight async execution, unified JavaScript/TypeScript tooling, and rich ecosystem for queue processing (BullMQ).
3. **Frontend Stack (React 18+ Vite + Zustand + React Query)**: Selected over Next.js/Vue. React 18 provides component-driven UI excellence, Vite offers sub-second HMR development speed, Zustand manages local client state without boilerplate, and TanStack React Query handles server-state caching and invalidation automatically.
4. **Connection Pooling & Caching (PgBouncer + Redis 7+)**: PgBouncer manages database connections in transaction mode to handle thousands of concurrent tenants efficiently. Redis 7+ serves as atomic token counter, real-time Socket.io Pub/Sub backplane, BullMQ job queue, and token blacklist store.

---

# SECTION 2: MOSCOW FEATURE PRIORITIZATION

```
+-----------------------------------------------------------------------------------+
|  healthcare+ Feature Prioritization Matrix                                        |
+-----------------------------------------------------------------------------------+
|  MUST HAVE (Phase 1-4 Core)                                                       |
|  - PostgreSQL RLS Multi-Tenancy & Parameterized Context Middleware                 |
|  - Argon2id Auth, RS256 JWT Rotation, Token Family Revocation & TOTP MFA          |
|  - Universal Healthcare Passport, ABHA ID Linkage & DPDP 2023 Soft Anonymization   |
|  - Doctor Multi-Hospital Affiliations & Department Schema                         |
|  - Idempotent Appointment Booking & Atomic Redis Queue Token Allocation           |
|  - PostGIS Emergency SOS, Atomic Driver Lock & 108/102 Tele-Dispatch Fallback     |
|  - Hybrid CDS Engine (Deterministic Core Gate + LLM Advisory Summaries)           |
|  - Pharmacy Order Fulfillment & Laboratory PDF Report Upload (ClamAV Scanned)     |
|  - Itemized Billing & Ed25519 Cryptographic Receipt Signatures                    |
+-----------------------------------------------------------------------------------+
|  SHOULD HAVE (Phase 4-5 Enhancements)                                             |
|  - Lite Appointments (Fractional Token Insertion e.g. Token #15.5)                |
|  - AI Crowd Status Computation Engine (🟢 Low, 🟡 Moderate, 🔴 High)               |
|  - Automated Medicine Reminders & Dose Logs (`dose_logs` table)                   |
|  - AI Symptom Triage Input Banner (Natural Language -> Department Mapping)        |
|  - Persistent In-App Notifications Feed (`notifications` table)                   |
+-----------------------------------------------------------------------------------+
|  COULD HAVE (Phase 6 Polish)                                                      |
|  - Dark / Light Mode Theme Switcher with CSS Tokens                               |
|  - Advanced Hospital Explorer Search & Filter Grid                                |
|  - Doctor Rating & Patient Feedback Module                                        |
+-----------------------------------------------------------------------------------+
|  FUTURE SCOPE (Post-Launch Expansion)                                             |
|  - WebRTC Peer-to-Peer Telemedicine Video Consultation                            |
|  - Bluetooth LE IoT Vitals Streaming (Pulse Oximeter, Smart Watch)                |
|  - ABDM FHIR Health Data Exchange Interoperability                                |
+-----------------------------------------------------------------------------------+
```

---

# SECTION 3: COMPLETE DATABASE DESIGN & PRODUCTION DDL SQL (v4.0)

```sql
-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";

-- 1. TENANTS TABLE (Hospital SaaS Tenants)
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) UNIQUE NOT NULL,
    address TEXT NOT NULL,
    location GEOMETRY(Point, 4326),
    contact_email VARCHAR(255) NOT NULL,
    contact_phone VARCHAR(50) NOT NULL,
    signing_public_key TEXT NOT NULL, -- Ed25519 Public Key for Digital Receipts
    is_active BOOLEAN DEFAULT TRUE,    -- Soft deletion path
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_tenants_location ON tenants USING GIST(location);

-- 1a. TENANT SETTINGS (Branding, Hours, Capacity)
CREATE TABLE tenant_settings (
    tenant_id UUID PRIMARY KEY REFERENCES tenants(id) ON DELETE CASCADE,
    logo_url TEXT,
    primary_color VARCHAR(20),
    working_hours JSONB DEFAULT '{}'::jsonb,
    daily_token_capacity_per_doctor INT DEFAULT 40,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 2. USERS & ROLES TABLE
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
    mfa_secret_encrypted TEXT, -- AES-256-GCM Encrypted Secret
    mfa_secret_iv VARCHAR(32), -- Initialization Vector
    is_mfa_enabled BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_tenant_role ON users(tenant_id, role);

-- 3. DEPARTMENTS TABLE
CREATE TABLE departments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, name)
);

-- 4. DOCTOR MULTI-HOSPITAL AFFILIATIONS
CREATE TABLE doctor_affiliations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    doctor_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    department_id UUID NOT NULL REFERENCES departments(id),
    consultation_fee NUMERIC(10,2) NOT NULL DEFAULT 500.00,
    lite_consultation_fee NUMERIC(10,2) NOT NULL DEFAULT 200.00,
    is_primary BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(doctor_id, tenant_id)
);

-- 5. HEALTHCARE PASSPORTS (Universal Identity & DPDP Compliance)
CREATE TABLE patient_passports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    patient_id UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    abha_id VARCHAR(50) UNIQUE, -- ABDM National Health ID
    blood_group VARCHAR(10),
    allergies JSONB DEFAULT '[]'::jsonb,
    chronic_conditions JSONB DEFAULT '[]'::jsonb,
    emergency_contact JSONB NOT NULL,
    is_anonymized BOOLEAN DEFAULT FALSE, -- DPDP Act 2023 Right to Erasure support
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 6. CONSENT GRANTS
CREATE TABLE passport_consent_grants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    passport_id UUID NOT NULL REFERENCES patient_passports(id) ON DELETE CASCADE,
    grantee_doctor_id UUID NOT NULL REFERENCES users(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    grant_type VARCHAR(50) NOT NULL, -- 'APPOINTMENT_AUTO' | 'EMERGENCY_OVERRIDE' | 'MANUAL_GRANT'
    granted_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ
);

CREATE INDEX idx_consent_active ON passport_consent_grants(passport_id, grantee_doctor_id) WHERE revoked_at IS NULL;

-- 7. APPOINTMENTS & QUEUE TABLE (With Lite Appointment Support)
CREATE TYPE appointment_type AS ENUM ('REGULAR', 'LITE_FOLLOWUP');
CREATE TYPE appointment_status AS ENUM ('PENDING_PAYMENT', 'CONFIRMED', 'IN_PROGRESS', 'COMPLETED', 'NO_SHOW', 'CANCELLED');

CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE RESTRICT,
    patient_id UUID NOT NULL REFERENCES users(id),
    doctor_id UUID NOT NULL REFERENCES users(id),
    appointment_date DATE NOT NULL,
    slot_time TIME NOT NULL,
    token_number NUMERIC(6,1) NOT NULL, -- Numeric supporting fractional tokens e.g. 15.5
    type appointment_type DEFAULT 'REGULAR',
    status appointment_status DEFAULT 'PENDING_PAYMENT',
    consultation_fee NUMERIC(10,2) NOT NULL,
    idempotency_key VARCHAR(255) UNIQUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, doctor_id, appointment_date, token_number)
);

CREATE INDEX idx_appointments_queue ON appointments(tenant_id, doctor_id, appointment_date, status);

-- Trigger checking doctor-tenant affiliation integrity
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

-- 8. PRESCRIPTIONS & MEDICINES
CREATE TABLE prescriptions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    appointment_id UUID REFERENCES appointments(id),
    patient_id UUID NOT NULL REFERENCES users(id),
    doctor_id UUID NOT NULL REFERENCES users(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    diagnosis TEXT NOT NULL,
    instructions TEXT,
    cds_evaluation JSONB, -- Stores Hybrid CDS Engine Result Audit Trail
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

-- 9. PHARMACY ORDERS & DOSE LOGS
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

CREATE TABLE dose_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    prescription_medicine_id UUID NOT NULL REFERENCES prescription_medicines(id) ON DELETE CASCADE,
    patient_id UUID NOT NULL REFERENCES users(id),
    scheduled_for TIMESTAMPTZ NOT NULL,
    taken_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_dose_logs_pending ON dose_logs(patient_id, scheduled_for) WHERE taken_at IS NULL;

-- 10. LABORATORY REQUESTS
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

-- 11. BILLING RECEIPTS & ITEMIZED ITEMS
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

-- 12. AMBULANCE FLEET & EMERGENCY DISPATCH
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

-- 13. AUDIT LOGS, NOTIFICATION CONSENT & IN-APP NOTIFICATIONS
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    action VARCHAR(100) NOT NULL,
    actor_user_id UUID REFERENCES users(id),
    target_passport_id UUID REFERENCES patient_passports(id),
    metadata JSONB DEFAULT '{}'::jsonb,
    ip_address INET,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE notification_consents (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id),
    channel VARCHAR(20) NOT NULL,
    consented BOOLEAN NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id),
    tenant_id UUID REFERENCES tenants(id),
    category VARCHAR(50) NOT NULL,
    title VARCHAR(255) NOT NULL,
    body TEXT,
    is_read BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_notifications_unread ON notifications(user_id, is_read) WHERE is_read = FALSE;

-- ROW LEVEL SECURITY (RLS) POLICIES ON ALL TENANT-SCOPED TABLES
DO $$
DECLARE
  t TEXT;
BEGIN
  FOREACH t IN ARRAY ARRAY[
    'users', 'departments', 'doctor_affiliations', 'appointments',
    'prescriptions', 'pharmacy_orders', 'lab_requests', 'billing_receipts',
    'ambulance_units', 'passport_consent_grants'
  ]
  LOOP
    EXECUTE format('ALTER TABLE %I ENABLE ROW LEVEL SECURITY;', t);
    EXECUTE format(
      'CREATE POLICY tenant_isolation_%1$I ON %1$I
         FOR ALL USING (
           tenant_id = NULLIF(current_setting(''app.current_tenant_id'', true), '''')::uuid
           OR current_setting(''app.current_tenant_id'', true) IS NULL
         );', t
    );
  END LOOP;
END $$;

ALTER TABLE emergency_dispatches ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_emergency_dispatches ON emergency_dispatches
    FOR ALL USING (
      assigned_tenant_id = NULLIF(current_setting('app.current_tenant_id', true), '')::uuid
      OR assigned_tenant_id IS NULL
    );
```

---

# SECTION 4: BACKEND DEEP-DIVE IMPLEMENTATION PLAN

## 4.1 Tenant Context Middleware & PgBouncer Query Wrapper
```javascript
// backend/src/middleware/tenant.middleware.js
export const enforceTenantContext = async (req, res, next) => {
  try {
    let tenantId = null;
    if (req.user && req.user.role === 'super_admin') {
      const headerTenant = req.headers['x-tenant-id'];
      if (headerTenant) {
        const tenant = await tenantRepository.findActiveById(headerTenant);
        if (!tenant) return res.status(404).json({ error: 'Tenant context invalid or deactivated' });
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

// backend/src/repository/withTenantContext.js
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

## 4.2 Idempotency Guard Middleware
```javascript
// backend/src/middleware/idempotency.middleware.js
export const idempotencyGuard = (resourceCheckFn) => async (req, res, next) => {
  const key = req.headers['idempotency-key'];
  if (!key) return res.status(400).json({ error: 'Idempotency-Key header required' });

  const existing = await resourceCheckFn(key);
  if (existing) {
    return res.status(200).json({ idempotent: true, data: existing });
  }
  req.idempotencyKey = key;
  next();
};
```

## 4.3 Atomic Queue & Lite Token Engine
```javascript
// backend/src/modules/queue/queue.service.js
export class QueueService {
  static async allocateRegularToken(tenantId, doctorId, date) {
    const redisKey = `token_counter:${tenantId}:${doctorId}:${date}`;
    const tokenNumber = await redis.incr(redisKey);
    await redis.expire(redisKey, 172800); // 48h TTL
    return tokenNumber;
  }

  static async allocateLiteToken(tenantId, doctorId, date) {
    const lockKey = `lite_lock:${tenantId}:${doctorId}:${date}`;
    const lock = await redis.set(lockKey, '1', 'NX', 'EX', 5);
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

        while (await slotTaken(client, tenantId, doctorId, date, candidate)) {
          candidate = Math.round((candidate + 0.1) * 10) / 10;
        }
        return candidate;
      });
    } finally {
      await redis.del(lockKey);
    }
  }
}
```

## 4.4 PostGIS Emergency Dispatch & Atomic Driver Lock Engine
```javascript
// backend/src/modules/emergency/emergency.service.js
export class EmergencyService {
  static async acceptDispatch(dispatchId, driverId, tenantId) {
    const { rows } = await pool.query(
      `UPDATE emergency_dispatches
       SET status = 'ACCEPTED', assigned_driver_id = $1, assigned_tenant_id = $2, updated_at = NOW()
       WHERE id = $3 AND status = 'BROADCASTING'
       RETURNING *;`,
      [driverId, tenantId, dispatchId]
    );

    if (rows.length === 0) {
      return { accepted: false, reason: 'ALREADY_ASSIGNED_OR_CLOSED' };
    }

    await ambulanceRepository.markUnavailable(driverId);
    socketService.emitToRoom(`emergency:${dispatchId}`, 'dispatch:accepted', rows[0]);
    return { accepted: true, dispatch: rows[0] };
  }

  static async triggerEmergency(patientId, location) {
    const candidates = await ambulanceRepository.findNearby(location, 10000); // 10km search
    const dispatch = await emergencyRepository.create({ patientId, location, candidates });

    if (candidates.length === 0) {
      // Immediate 108/102 Escalation
      await emergencyRepository.escalateToExternalDispatch(dispatch.id);
      return dispatch;
    }

    socketService.broadcastToDrivers(candidates, dispatch);
    return dispatch;
  }
}
```

## 4.5 Hybrid Clinical Decision Support (CDS) Engine
```javascript
// backend/src/modules/triage/cds.service.js
export class HybridCDSEngine {
  static async evaluatePrescription(patientPassport, newMedicines) {
    const hardWarnings = [];

    // 1. DETERMINISTIC SAFETY RULES (Primary Blocking Gate)
    for (const med of newMedicines) {
      if (patientPassport.age < 12 && ['aspirin', 'tetracycline'].some(d => med.name.toLowerCase().includes(d))) {
        hardWarnings.push(`CRITICAL CDS ALERT: ${med.name} is contraindicated in pediatric patients (<12 years).`);
      }
      if (patientPassport.allergies.some(a => med.name.toLowerCase().includes(a.toLowerCase()))) {
        hardWarnings.push(`CRITICAL ALLERGY ALERT: Patient is allergic to ${med.name}.`);
      }
      if (parseInt(med.dosagePerDay) > 4) {
        hardWarnings.push(`DOSAGE ALERT: High daily frequency (${med.dosagePerDay}x/day) for ${med.name}.`);
      }
    }

    if (hardWarnings.length > 0) {
      return { status: 'REJECTED', warnings: hardWarnings, source: 'DETERMINISTIC_ENGINE' };
    }

    // 2. LLM AI EXPLANATORY LAYER (Secondary Advisory)
    const aiSummary = await AIService.generateMedicationSummary(newMedicines);
    return { status: 'APPROVED', warnings: [], aiSummary, source: 'HYBRID_ENGINE' };
  }
}
```

## 4.6 MFA Secret AES-256-GCM Encryption Helper
```javascript
// backend/src/utils/mfaCrypto.js
import crypto from 'crypto';
const ALGO = 'aes-256-gcm';
const KEY = Buffer.from(process.env.MFA_ENCRYPTION_KEY, 'hex');

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

## 4.7 Socket.io Handshake Auth & Room Authorization
```javascript
// backend/src/services/socket.authorization.js
export async function authorizeRoomAccess(user, room) {
  const [type, ...rest] = room.split(':');

  if (type === 'queue') {
    const [tenantId, doctorId] = rest;
    if (user.role === 'patient') {
      return appointmentRepository.hasActiveAppointment(user.id, doctorId, tenantId);
    }
    return user.tenantId === tenantId;
  }

  if (type === 'emergency') {
    const [dispatchId] = rest;
    const dispatch = await emergencyRepository.findById(dispatchId);
    if (!dispatch) return false;
    return (
      dispatch.patient_id === user.id ||
      dispatch.assigned_driver_id === user.id ||
      (user.tenantId && user.tenantId === dispatch.assigned_tenant_id)
    );
  }

  return false;
}
```

---

# SECTION 5: FRONTEND DEEP-DIVE IMPLEMENTATION PLAN

## 5.1 Design Tokens & CSS Architecture (`src/styles/tokens.css`)
```css
:root {
  --bg: #060b14; --s1: #0b1220; --s2: #101829; --s3: #162035;
  --b1: #1a2840; --b2: #1f3050; --b3: #253860;
  --c1: #00f5d4; --c2: #3b9eff; --c3: #f0a500; --c4: #f43f5e; --c5: #a78bfa;
  --tx: #dce8f5; --tx2: #7b98bc; --tx3: #3d5a7a;
  --f: 'Outfit', sans-serif; --m: 'Fira Code', monospace;
  --r: 10px; --r2: 16px; --r3: 20px;
}
:root.light-mode {
  --bg: #f1f5f9; --s1: #ffffff; --s2: #f8fafc; --s3: #eff6ff;
  --b1: #e2e8f0; --b2: #cbd5e1; --b3: #94a3b8;
  --tx: #1e293b; --tx2: #475569; --tx3: #94a3b8;
}
```

## 5.2 User Home Dashboard Wireframe & Flow
```
+-----------------------------------------------------------------------------------+
|  healthcare+  |  Passport ID: HP-9082-331  |  [ Nearby ] [ Personal ] (🔔 3) (👤)  |
+-----------------------------------------------------------------------------------+
|                                                                                   |
|  🤖 AI Health Assistant (Top Banner)                                              |
|  +-----------------------------------------------------------------------------+  |
|  |  "How are you feeling today?"                                               |  |
|  |  [ E.g., Fever for the last 5 days                                        ] |  |
|  |  < Analyze & Suggest Department >                                           |  |
|  |  Suggested: General Physician  -->  < View Available Doctors >               |  |
|  +-----------------------------------------------------------------------------+  |
|                                                                                   |
|  🚨 Emergency SOS Button (Always Visible)                                         |
|  +-----------------------------------------------------------------------------+  |
|  |  [ PRESS & HOLD 3 SECONDS FOR AMBULANCE SOS ]                                |  |
|  +-----------------------------------------------------------------------------+  |
|                                                                                   |
|  🏥 Nearby Subscribed Hospitals (Nearest 4 Displayed)                              |
|  +-----------------------------------------------------------------------------+  |
|  | City General Hospital (1.2 km)  ⭐ 4.8  | Crowd Status: 🟢 Low   | Fee: ₹500    |  |
|  | Depts: Cardiology, Neurology, General Medicine                             |  |
|  | < Book Appointment >                                                        |  |
|  +-----------------------------------------------------------------------------+  |
|  | Apollo Care Center (3.5 km)      ⭐ 4.6  | Crowd Status: 🟡 Moderate | Fee: ₹600 |  |
|  | Depts: General, Pediatrics, Dermatology                                     |  |
|  | < Book Appointment >                                                        |  |
|  +-----------------------------------------------------------------------------+  |
+-----------------------------------------------------------------------------------+
```

## 5.3 Emergency SOS Uber/Rapido Dispatch Interface
1. Hold SOS button for 3 seconds → Confirmation Dialog (*"Are you sure?"*).
2. On Confirmation: Browser captures GPS coordinates → Transits to dispatch UI:
   - State 1: `Searching for nearby ambulances...`
   - State 2: `Driver Accepted! Driver: Ramesh Kumar (KA-01-EA-1234)`
   - State 3: `Live Map GPS Tracking & Dynamic ETA (e.g., 8 mins)`
   - State 4: `Destination Hospital ER Pre-Notified`

## 5.4 Patient Dashboard & Hospital Workspace Navigation
- **Patient Dashboard**: Toggle between Main Dashboard and Personal Dashboard (Upcoming appointments, live token ticket, active prescriptions with `dose_logs` checkboxes, lab report PDFs, billing receipts, Healthcare Passport timeline).
- **Hospital Workspace**: Isolated hospital ecosystem (`Doctors`, `Appointments`, `Pharmacy`, `Laboratory`, `Billing`, `Notifications`). Booking flow supports regular booking and **Lite Appointment Toggle** (token `#15.5`).

---

# SECTION 6: API SPECIFICATION MATRIX

| Method | Endpoint | Auth Level | Request Body / Params | Expected Response | SLA Tier |
|:---|:---|:---|:---|:---|:---|
| `POST` | `/api/v1/auth/register` | Public | `{ email, password, full_name, role }` | `{ user, token }` | < 100ms |
| `POST` | `/api/v1/auth/login` | Public | `{ email, password }` | `{ user, accessToken, refreshToken }` | < 100ms |
| `POST` | `/api/v1/auth/refresh` | Refresh Token | `{ refreshToken }` | `{ accessToken, refreshToken }` | < 100ms |
| `POST` | `/api/v1/auth/mfa/verify` | Session Token | `{ code }` | `{ user, accessToken }` | < 100ms |
| `GET` | `/api/v1/hospitals/nearby` | Authenticated | `?lat=12.97&lng=77.59&radius=10` | `[{ id, name, distance, crowdStatus }]` | < 100ms |
| `POST` | `/api/v1/triage/analyze` | Authenticated | `{ symptoms: "fever 5 days" }` | SSE Stream `{ department: "General" }` | < 3.0s |
| `POST` | `/api/v1/appointments/book` | Patient | Header: `Idempotency-Key` | `{ appointmentId, tokenNumber: 15 }` | < 100ms |
| `POST` | `/api/v1/appointments/book-lite`| Patient | Header: `Idempotency-Key` | `{ appointmentId, tokenNumber: 15.5 }` | < 100ms |
| `GET` | `/api/v1/queue/live/:doctorId` | Authenticated | `?date=2026-03-01` | `{ currentToken: 14, waitMins: 15 }` | < 100ms |
| `POST` | `/api/v1/emergency/trigger` | Patient | `{ lat: 12.97, lng: 77.59 }` | `{ dispatchId, status: "BROADCASTING" }`| < 100ms |
| `POST` | `/api/v1/emergency/accept/:id` | Driver | `{}` | `{ accepted: true, dispatch }` | < 50ms |
| `GET` | `/api/v1/passport/timeline` | Auth + Consent | `{}` | `{ passportId, timeline: [...] }` | < 100ms |
| `POST` | `/api/v1/prescriptions/sign` | Doctor | `{ appointmentId, medicines }` | `{ prescriptionId, cdsEvaluation }` | < 100ms |
| `POST` | `/api/v1/pharmacy/orders/:id/confirm`| Patient | `{}` | `{ orderId, status: "RECEIVED" }` | < 100ms |
| `POST` | `/api/v1/lab/reports/upload` | Lab Tech | FormData: `file` | `{ reportUrl, status: "COMPLETED" }` | < 5.0s |
| `PATCH`| `/api/v1/dose-logs/:id/taken` | Patient | `{}` | `{ doseLogId, takenAt: "..." }` | < 100ms |

---

# SECTION 7: ROLE-BASED WORKFLOWS & STATE MACHINES

## 7.1 Emergency SOS State Machine
```
IDLE
  → (hold 3s) → CONFIRM_MODAL
      → (confirm) → LOCATING            [HTML5 Geolocation]
          → (coords acquired) → BROADCASTING   [PostGIS 10km search]
              → (0 candidates) → ESCALATED_108   [terminal state]
              → (≥1 candidate) → candidates notified via socket
                  → (driver accepts, atomic lock wins) → ACCEPTED
                      → EN_ROUTE_PATIENT → PATIENT_PICKED → ARRIVED_HOSPITAL  [terminal]
                  → (15s no acceptance) → ESCALATED_108   [terminal]
      → (cancel before driver accepts) → CANCELLED   [terminal]
```

## 7.2 Pharmacy Order State Machine
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

## 7.3 Laboratory Request State Machine
```
Doctor creates request [lab_requests: REQUESTED]
   → Patient views cost, confirms & pays → PAID
       → Lab marks SAMPLE_COLLECTED
           → Lab marks PROCESSING
               → Lab uploads PDF (ClamAV scan + magic-byte check) → COMPLETED
                   → report_file_url attached
                   → in-app notification fired to both doctor and patient
                   → visible immediately in Healthcare Passport timeline
```

---

# SECTION 8: SECURITY, COMPLIANCE & SCALABILITY STRATEGY

1. **OWASP Top 10 Hardening**:
   - **SQL Injection**: Parameterized SQL queries and `set_config($1, $2, true)` for RLS context.
   - **XSS & CORS**: Helmet.js security headers, CORS origin locking, and HTML input sanitization.
   - **Rate Limiting**: `express-rate-limit` enforcing 100 req/min per IP, 5 req/min on auth endpoints.
2. **Identity & Data Encryption**:
   - **Password Hashing**: Argon2id with unique per-user salt.
   - **MFA Encryption**: AES-256-GCM application-layer encryption for `users.mfa_secret_encrypted`.
   - **Digital Receipts**: Ed25519 asymmetric signatures verified via tenant public keys.
3. **India DPDP Act 2023 Compliance**:
   - **Right to Erasure**: Soft anonymization via `is_anonymized = true`, scrubbing PII while retaining statutory medical and billing audit logs.
   - **ABHA Linkage**: `abha_id` field on `patient_passports` supporting national ABDM Health ID integration.
   - **Consent Ledger**: Opt-in/opt-out preferences stored in `notification_consents` per TRAI regulations.
4. **Scalability Architecture**:
   - **Stateless Application Cluster**: Express app instances managed via PM2 behind NGINX.
   - **Connection Pooling**: PgBouncer managing PostgreSQL connection limits.
   - **Redis Caching**: Redis 7+ caching hospital crowd status, user sessions, and Pub/Sub events.

---

# SECTION 9: EDGE CASES & FAILURE MODES MATRIX

| Failure Scenario | Root Cause | System Resolution / Failover Strategy |
|:---|:---|:---|
| **Redis cluster failure** | Redis node crash | System falls back to Postgres `SELECT FOR UPDATE` atomic token locking (§4.3); if Postgres is degraded, returns HTTP 503 instead of issuing duplicate tokens. |
| **Doctor deactivated mid-day** | Staff leave / emergency | Existing `CONFIRMED` tokens remain visible with a "Doctor Unavailable" banner; no auto-cancellation, requiring manual reception queue triage. |
| **Lite token collision** | Concurrent lite bookings | Redis lock `lite_lock:{tenant}:{doctor}:{date}` holds execution; candidate half-slot steps forward in 0.1 increments until a free slot is acquired. |
| **Consent revoked mid-consult** | Patient revokes access | Access remains granted for the active consultation until appointment reaches `COMPLETED`, avoiding abrupt interruption during in-progress care. |
| **Driver loses GPS connectivity**| Mobile network drop | Dispatch room alerts hospital ER console; system triggers automatic SMS backup alert to driver's mobile number. |
| **Lab PDF fails ClamAV scan** | Malware / corrupted PDF | File upload rejected before storage; lab tech receives security alert; lab request status remains `PROCESSING`. |

---

# SECTION 10: REALISTIC DEVELOPMENT ROADMAP & TASK BREAKDOWN

```
Phase 1: Architecture & Database Foundations (Weeks 1-2)
├── Task 1.1: Express Node.js ESM repository setup with Pino logging & Zod env validation.
├── Task 1.2: PostgreSQL 16 + PostGIS DDL migration script (15 tables, indexes, triggers).
├── Task 1.3: RLS policy application across all 12 operational tables.
└── Task 1.4: Parameterized tenant middleware & PgBouncer withTenantContext transaction wrapper.

Phase 2: Auth, Security & Healthcare Passport (Weeks 3-4)
├── Task 2.1: Argon2id password hashing, RS256 JWT rotation & Redis token family revocation.
├── Task 2.2: AES-256-GCM MFA secret encryption & TOTP verification endpoints.
├── Task 2.3: Healthcare Passport APIs, ABHA ID linkage & DPDP 2023 soft anonymization handler.
└── Task 2.4: Passport consent grant matrix (APPOINTMENT_AUTO, EMERGENCY_OVERRIDE, MANUAL_GRANT).

Phase 3: Multi-Hospital Engine, Queue & Lite Appointments (Weeks 5-6)
├── Task 3.1: Hospital tenant onboarding, department setup & doctor_affiliations APIs.
├── Task 3.2: Atomic Redis INCR token allocation engine & Postgres FOR UPDATE fallback.
├── Task 3.3: Lite Appointment fractional token generator (Token #15.5) with Redis locks.
└── Task 3.4: AI Crowd Status engine (computeCrowdStatus) with 60s Redis caching.

Phase 4: AI Triage & Emergency SOS Dispatch (Weeks 7-8)
├── Task 4.1: Gemini AI Symptom Triage service with natural language to department JSON mapping.
├── Task 4.2: PostGIS ST_DWithin spatial ambulance search & zero-driver 108/102 fallback.
├── Task 4.3: Atomic driver acceptance UPDATE...RETURNING execution & socket broadcast.
└── Task 4.4: Authenticated Socket.io server with per-room authorization guard (authorizeRoomAccess).

Phase 5: Clinical Workflows, Lab, Pharmacy & Billing (Weeks 9-10)
├── Task 5.1: Hybrid CDS Engine (Deterministic safety gate + LLM advisory summaries).
├── Task 5.2: Pharmacy fulfillment order workflow & BullMQ scheduled medicine reminder logs (dose_logs).
├── Task 5.3: Laboratory sample tracking workflow & ClamAV scanned PDF report uploader.
└── Task 5.4: Standalone & appointment billing receipts with Ed25519 digital signatures.

Phase 6: Frontend UI Component Library & Dashboards (Weeks 11-12)
├── Task 6.1: Vite + React 18 design token system, theme switcher & primitive components.
├── Task 6.2: User Home Dashboard (AI Triage top banner, hospital search, 4-nearest hospital cards, SOS modal).
├── Task 6.3: Personal Patient Dashboard, active queue ticket widget, dose_logs checkboxes, Passport timeline.
└── Task 6.4: Hospital Workspace, doctor booking wizard with Lite Appointment toggle, live queue ticker.

Phase 7: Consoles, Security Audit & Deployment (Weeks 13-14)
├── Task 7.1: Doctor Desk with CDS warning banners, Pharmacy console & Lab report uploader UI.
├── Task 7.2: Hospital Admin Panel & Platform Super Admin Console.
├── Task 7.3: Cross-tenant RLS penetration test suite & K6 load testing under PgBouncer concurrency.
└── Task 7.4: NGINX reverse proxy, PM2 cluster, Docker Compose / K8s production rollout.
```

---

# SECTION 11: FINAL PRODUCTION READINESS VERIFICATION CHECKLIST

- [ ] **Database & RLS Integrity**: All 15 PostgreSQL tables deployed; RLS policies active and verified via automated cross-tenant isolation test suite.
- [ ] **Tenant Middleware Security**: `x-tenant-id` header untrusted for non-super-admins; `set_config($1, $2, true)` parameterized; `withTenantContext` transaction wrapper enforced across all repository queries.
- [ ] **Auth & Encryption**: Argon2id password hashing verified; RS256 JWT rotation with Redis token family revocation functional; `users.mfa_secret_encrypted` encrypted via AES-256-GCM.
- [ ] **Idempotency Protection**: Booking endpoints enforce `Idempotency-Key` header, preventing duplicate appointments or billing charges.
- [ ] **Atomic Queue Engine**: Redis `INCR` counter with Postgres `SELECT FOR UPDATE` fallback verified under simulated 2,000 req/sec booking load.
- [ ] **Lite Appointments**: Fractional token generation (`15.5`) verified under concurrent lite booking tests.
- [ ] **Emergency Dispatch System**: PostGIS spatial search verified; atomic driver acceptance (`UPDATE...RETURNING`) confirmed; zero-driver radius verified to trigger immediate 108/102 escalation.
- [ ] **Hybrid CDS Engine**: Deterministic core rule engine unit-tested against pediatric contraindications and drug allergy test vectors before AI layer sign-off.
- [ ] **Pharmacy & Reminders**: Order fulfillment status stepper verified; `dose_logs` table populates scheduled reminder rows upon order completion.
- [ ] **Laboratory Pipeline**: PDF report upload pipeline scans files via ClamAV and verifies magic-bytes before attaching to Passport.
- [ ] **DPDP Act 2023 Compliance**: Patient right-to-erasure soft anonymization verified; ABHA ID linkage functional; notification consent ledger enforced.
- [ ] **Real-Time Security**: Socket.io connection handshake authenticates JWT; `authorizeRoomAccess` guard rejects unauthorized room joins.
- [ ] **Production Infrastructure**: NGINX SSL/TLS reverse proxy, PM2 process cluster, Redis Pub/Sub, PgBouncer transaction pooling, and automated backup cron jobs verified.
