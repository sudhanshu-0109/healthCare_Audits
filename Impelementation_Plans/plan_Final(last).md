# healthcare+ — Master Implementation Plan & Production Architecture (v2.0 Final)

> **Document Status**: ARCHITECTURE APPROVED & BUILD-READY.
> **Target Stack**: PERN (PostgreSQL 16+ with PostGIS, Express.js, React 18+ Vite, Node.js 20+ LTS).
> **Product Vision**: Multi-Hospital SaaS Healthcare Operating System & Universal Patient Healthcare Passport.
> **Compliance Target**: India Digital Personal Data Protection (DPDP) Act 2023, TRAI SMS Regulations, and ABDM ABHA Standard.

---

## 0. Version Change Log (v1.0 Draft vs. v2.0 Final)

| # | Feature / Area | v1.0 Draft Deficit | v2.0 Final Specification |
|:---|:---|:---|:---|
| 1 | **Tenant Middleware** | Trusted client header `x-tenant-id` & raw SQL interpolation | Context derived strictly from JWT claim (`req.user.tenantId`) for non-super-admins; parameterized `set_config` |
| 2 | **RLS + PgBouncer** | `SET LOCAL` leaking under transaction pooling | Explicit `withTenantContext` transaction wrapper per tenant-scoped database request |
| 3 | **Billing Schema** | Missing FK to appointments; contradicted standalone lab billing | Made `appointment_id` optional; added `lab_test_id` and itemized `receipt_items` table |
| 4 | **Queue Tokens** | Race condition under concurrent bookings | Atomic Redis `INCR` counter per doctor/date with Postgres `SELECT FOR UPDATE` fallback |
| 5 | **Ambulance Lock** | Non-atomic driver acceptance | Atomic conditional update (`UPDATE...WHERE status='BROADCASTING' RETURNING *`) |
| 6 | **Ambulance Fallback** | 15s driver timeout assuming candidates exist | Zero-candidate PostGIS spatial query triggers immediate 108/102 tele-dispatch escalation |
| 7 | **CDS Engine** | Probabilistic LLM replacing deterministic checks | **Hybrid CDS Engine**: Deterministic core engine primary & blocking; LLM advisory only |
| 8 | **Doctor Model** | Assumed 1 doctor = 1 hospital tenant | Added `doctor_affiliations` join table supporting multi-hospital practices |
| 9 | **NFR SLA Tiers** | Flat <100ms target conflicting with LLM latency | Tiered SLA: <100ms REST, <50ms WebSockets, <3s AI Triage (SSE/Streaming) |
| 10 | **Compliance Regime**| Standard US HIPAA/EU GDPR focus | **DPDP Act 2023 Primary**: Consent ledger, right-to-erasure via anonymization, ABHA ID linkage |
| 11 | **Security & Auth** | No MFA, no token reuse detection, raw file uploads | TOTP MFA mandatory for staff; refresh token family reuse detection; ClamAV scan for PDF uploads |
| 12 | **Real-Time Auth** | Missing WebSocket room authorization | JWT handshake authentication + per-room entitlement check (`authorizeRoomAccess`) |

---

## 1. System Architecture & Infrastructure Topology

```
                                  [ Cloudflare WAF / DNS ]
                                             |
                                             v
                                  [ NGINX Reverse Proxy ]
                                             |
                  +--------------------------+--------------------------+
                  |                                                     |
                  v                                                     v
      [ Express Node Cluster ]                              [ React Frontend CDN ]
      (PM2 Process Manager)                                 (Static Assets via Vite)
                  |
        +---------+---------+
        |                   |
        v                   v
[ PostgreSQL 16 Primary ]   [ Redis Cluster ]
(PostGIS + Read Replica)    (Cache + Pub/Sub + BullMQ + Token Counter)
        ^
        |
   [ PgBouncer ]
(Transaction Mode)
```

---

## 2. Multi-Tenancy & Secure RLS Context Engine

### 2.1 Tenant Isolation Model
The application operates on a single PostgreSQL database using logical tenant separation. `tenant_id` foreign keys are enforced across operational tables via PostgreSQL Row-Level Security (RLS). Global health records (Patients, Healthcare Passports, Emergency Dispatches) exist independently of hospital tenants.

### 2.2 Secure Tenant Context Middleware
Client-supplied `x-tenant-id` headers are **never** trusted for non-super-admin users.

```javascript
// backend/src/middleware/tenant.middleware.js
export const enforceTenant = async (req, res, next) => {
  const role = req.user?.role;
  let tenantId;

  if (role === 'super_admin') {
    const headerTenant = req.headers['x-tenant-id'];
    if (headerTenant) {
      const tenant = await tenantRepository.findActiveById(headerTenant);
      if (!tenant) return res.status(404).json({ error: 'Tenant context invalid or deactivated' });
      tenantId = tenant.id;
    }
  } else {
    // Non-super-admins strictly use the tenant ID signed inside their JWT
    tenantId = req.user?.tenantId;
    if (!tenantId && req.isTenantRoute) {
      return res.status(403).json({ error: 'Tenant context missing' });
    }
  }

  req.tenantId = tenantId ?? null;
  next();
};
```

### 2.3 PgBouncer Transaction-Pooling Safe RLS Wrapper

```javascript
// backend/src/repository/withTenantContext.js
export async function withTenantContext(tenantId, fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    // Parameterized configuration setting to prevent SQL injection
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

---

## 3. Database Schema (PostgreSQL 16 + PostGIS DDL v2.0)

```sql
-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";

-- 1. TENANTS TABLE (Hospital SaaS)
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) UNIQUE NOT NULL,
    address TEXT NOT NULL,
    location GEOMETRY(Point, 4326),
    contact_email VARCHAR(255) NOT NULL,
    contact_phone VARCHAR(50) NOT NULL,
    signing_public_key TEXT NOT NULL, -- Ed25519 public key for digital receipts
    is_active BOOLEAN DEFAULT TRUE,    -- Soft deletion path
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_tenants_location ON tenants USING GIST(location);

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
    mfa_secret VARCHAR(255),
    is_mfa_enabled BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_tenant_role ON users(tenant_id, role);

-- 3. DOCTOR MULTI-HOSPITAL AFFILIATIONS
CREATE TABLE doctor_affiliations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    doctor_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    specialty VARCHAR(100) NOT NULL,
    consultation_fee NUMERIC(10,2) NOT NULL DEFAULT 500.00,
    is_primary BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(doctor_id, tenant_id)
);

-- 4. HEALTHCARE PASSPORTS (Universal Identity & DPDP Compliance)
CREATE TABLE patient_passports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    patient_id UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    abha_id VARCHAR(50) UNIQUE, -- ABDM National Health ID linkage
    blood_group VARCHAR(10),
    allergies JSONB DEFAULT '[]'::jsonb,
    chronic_conditions JSONB DEFAULT '[]'::jsonb,
    emergency_contact JSONB NOT NULL,
    is_anonymized BOOLEAN DEFAULT FALSE, -- DPDP 2023 Right to Erasure support
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 5. CONSENT GRANTS & BREAK-GLASS AUDIT LOGS
CREATE TABLE consent_grants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    passport_id UUID NOT NULL REFERENCES patient_passports(id) ON DELETE CASCADE,
    grantee_doctor_id UUID NOT NULL REFERENCES users(id),
    grant_type VARCHAR(50) NOT NULL, -- 'CONSULTATION' | 'EMERGENCY_OVERRIDE' | 'MANUAL'
    granted_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ
);

CREATE INDEX idx_consent_active ON consent_grants(passport_id, grantee_doctor_id) WHERE revoked_at IS NULL;

-- 6. APPOINTMENTS & QUEUE TABLE
CREATE TYPE appointment_status AS ENUM (
    'PENDING_PAYMENT', 'CONFIRMED', 'IN_PROGRESS', 'COMPLETED', 'NO_SHOW', 'CANCELLED'
);

CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE RESTRICT,
    patient_id UUID NOT NULL REFERENCES users(id),
    doctor_id UUID NOT NULL REFERENCES users(id),
    appointment_date DATE NOT NULL,
    slot_time TIME NOT NULL,
    token_number INT NOT NULL,
    status appointment_status DEFAULT 'PENDING_PAYMENT',
    consultation_fee NUMERIC(10,2) NOT NULL,
    idempotency_key VARCHAR(255) UNIQUE, -- Double booking protection
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, doctor_id, appointment_date, token_number)
);

CREATE INDEX idx_appointments_queue ON appointments(tenant_id, doctor_id, appointment_date, status);

-- 7. BILLING RECEIPTS & ITEMIZED TRANSACTIONS (Standalone & Walk-In Billing)
CREATE TABLE billing_receipts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    patient_id UUID NOT NULL REFERENCES users(id),
    appointment_id UUID REFERENCES appointments(id) ON DELETE SET NULL,
    lab_test_id UUID REFERENCES lab_tests(id) ON DELETE SET NULL,
    receipt_number VARCHAR(100) UNIQUE NOT NULL,
    total_amount NUMERIC(10,2) NOT NULL,
    tax_amount NUMERIC(10,2) DEFAULT 0.00,
    discount_amount NUMERIC(10,2) DEFAULT 0.00,
    is_paid BOOLEAN DEFAULT FALSE,
    payment_method VARCHAR(50),
    transaction_ref VARCHAR(255),
    digital_signature TEXT NOT NULL, -- Ed25519 signature
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_billing_has_source CHECK (appointment_id IS NOT NULL OR lab_test_id IS NOT NULL)
);

CREATE TABLE receipt_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    receipt_id UUID NOT NULL REFERENCES billing_receipts(id) ON DELETE CASCADE,
    item_type VARCHAR(50) NOT NULL,
    reference_id UUID,
    description VARCHAR(255) NOT NULL,
    quantity INT NOT NULL DEFAULT 1,
    unit_price NUMERIC(10,2) NOT NULL,
    total_price NUMERIC(10,2) NOT NULL
);

-- 8. AMBULANCE FLEET & EMERGENCY DISPATCH
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
    broadcast_candidate_ids UUID[] DEFAULT '{}',
    eta_minutes INT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_emergency_location ON emergency_dispatches USING GIST(patient_location);

-- 9. AUDIT LOGS & NOTIFICATION CONSENT LEDGER
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
    channel VARCHAR(20) NOT NULL, -- 'SMS' | 'EMAIL' | 'PUSH'
    consented BOOLEAN NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- ROW LEVEL SECURITY (RLS) POLICIES
ALTER TABLE appointments ENABLE ROW LEVEL SECURITY;
ALTER TABLE billing_receipts ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_appointments ON appointments
    FOR ALL USING (tenant_id = NULLIF(current_setting('app.current_tenant_id', true), '')::uuid);

CREATE POLICY tenant_isolation_billing ON billing_receipts
    FOR ALL USING (tenant_id = NULLIF(current_setting('app.current_tenant_id', true), '')::uuid);
```

---

## 4. Concurrency Engines & Safety Workflows

### 4.1 Atomic Token Queue Generation
```javascript
// services/queue.service.js
async function allocateToken(tenantId, doctorId, date) {
  const redisKey = `token_counter:${tenantId}:${doctorId}:${date}`;
  const tokenNumber = await redis.incr(redisKey);
  await redis.expire(redisKey, 172800); // 2-day TTL

  try {
    await withTenantContext(tenantId, (client) =>
      appointmentRepository.insertAppointment(client, { tenantId, doctorId, date, tokenNumber })
    );
    return tokenNumber;
  } catch (err) {
    if (isUniqueViolation(err)) {
      // Fallback under row lock if Redis counter desynced
      return allocateTokenFallback(tenantId, doctorId, date);
    }
    throw err;
  }
}
```

### 4.2 Atomic Ambulance Lock & Zero-Candidate Fallback
```javascript
// services/emergency.service.js
async function acceptDispatch(dispatchId, driverId) {
  // Single atomic UPDATE execution prevents duplicate driver assignment
  const { rows } = await pool.query(
    `UPDATE emergency_dispatches
     SET status = 'ACCEPTED', assigned_driver_id = $1, updated_at = NOW()
     WHERE id = $2 AND status = 'BROADCASTING'
     RETURNING *`,
    [driverId, dispatchId]
  );

  if (rows.length === 0) {
    return { accepted: false, reason: 'ALREADY_ASSIGNED_OR_CLOSED' };
  }

  await ambulanceRepository.markUnavailable(driverId);
  socketService.emitToRoom(`emergency:${dispatchId}`, 'dispatch:accepted', rows[0]);
  return { accepted: true, dispatch: rows[0] };
}

async function triggerEmergency(patientId, location) {
  const candidates = await ambulanceRepository.findNearby(location, 10000); // 10km PostGIS search
  const dispatch = await emergencyRepository.create({ patientId, location, candidates });

  if (candidates.length === 0) {
    // Zero drivers found -> Immediate Escalation
    await emergencyRepository.escalateToExternalDispatch(dispatch.id); // 108/102 Tele-dispatch
    return dispatch;
  }

  socketService.broadcastToDrivers(candidates, dispatch);
  return dispatch;
}
```

### 4.3 Hybrid Clinical Decision Support (CDS) Engine
```javascript
// modules/triage/cds.service.js
export class HybridCDSEngine {
  static async evaluatePrescription(patientPassport, newMedicines) {
    const hardWarnings = [];

    // 1. DETERMINISTIC SAFETY RULES (Primary & Blocking Gate)
    for (const med of newMedicines) {
      if (patientPassport.age < 12 && ['aspirin', 'tetracycline'].some(d => med.name.toLowerCase().includes(d))) {
        hardWarnings.push(`CRITICAL CDS ALERT: ${med.name} is contraindicated in pediatric patients (<12 years).`);
      }
      if (patientPassport.allergies.some(a => med.name.toLowerCase().includes(a.toLowerCase()))) {
        hardWarnings.push(`CRITICAL ALLERGY ALERT: Patient is allergic to ${med.name}.`);
      }
    }

    if (hardWarnings.length > 0) {
      return { status: 'REJECTED', warnings: hardWarnings, source: 'DETERMINISTIC_ENGINE' };
    }

    // 2. LLM AI EXPLANATORY LAYER (Advisory Only)
    const aiSummary = await AIService.generateMedicationSummary(newMedicines);
    return { status: 'APPROVED', warnings: [], aiSummary, source: 'HYBRID_ENGINE' };
  }
}
```

---

## 5. Security & Real-Time Engine

1. **MFA (TOTP)**: Mandatory for `super_admin` and `hospital_admin` logins.
2. **Refresh Token Reuse Detection**: Single-use rotation tracking via `token_family_id` in Redis. Reuse of an old token immediately revokes the entire family.
3. **File Upload Security**: Lab PDFs validated via magic-bytes, scanned by ClamAV, capped at 10MB, and stored in non-executable S3 buckets with presigned URLs.
4. **WebSocket Room Authorization**:
```javascript
socket.on('join', async ({ room }) => {
  const isAuthorized = await authorizeRoomAccess(socket.user, room);
  if (!isAuthorized) return socket.emit('error', 'Unauthorized room access');
  socket.join(room);
});
```

---

## 6. Phased Development Roadmap

```
Phase 1: Foundations (Weeks 1-2)
- Tenant middleware, withTenantContext PgBouncer wrapper, database DDL & RLS deployment.

Phase 2: Auth & Healthcare Passport (Weeks 3-4)
- Argon2id, JWT rotation with token family revocation, TOTP MFA, ABHA ID linkage, DPDP anonymization.

Phase 3: Multi-Hospital & Queue Engine (Weeks 5-6)
- Doctor affiliations, atomic Redis INCR token allocation, live Socket.io queue rooms.

Phase 4: AI Triage & Emergency System (Weeks 7-8)
- Gemini AI Triage, Hybrid CDS Engine, PostGIS ambulance search, atomic driver lock & 108/102 escalation.

Phase 5: Lab, Pharmacy & Billing Integration (Weeks 9-10)
- Standalone billing receipts, Ed25519 signatures, BullMQ notifications, file upload scanner.

Phase 6: Security Audit, Testing & Production Deployment (Weeks 11-12)
- K6 load testing, cross-tenant RLS penetration test suite, Docker Compose / K8s production rollout.
```

---

## 7. Production Readiness Verification Checklist

- [ ] All PostgreSQL tables created with foreign keys, GIST indexes, and RLS policies passing automated cross-tenant isolation test suite.
- [ ] Tenant middleware parameterization (`set_config`) verified; zero raw SQL string interpolations.
- [ ] `withTenantContext` wrapper enforced across all repository calls via custom ESLint rule.
- [ ] Redis token counter + Postgres fallback verified under simulated concurrent booking load (K6).
- [ ] Emergency driver acceptance lock tested under concurrent driver acceptance calls.
- [ ] Zero-driver PostGIS search verified to trigger immediate 108/102 gateway escalation.
- [ ] Deterministic CDS safety rules unit-tested against known pediatric and drug allergy test vectors.
- [ ] Refresh token family revocation verified upon token replay attempt.
- [ ] Socket.io room authorization middleware verified to reject unauthorized patient/staff room joins.
- [ ] DPDP Act 2023 anonymization-on-erasure workflow verified with compliance audit sign-off.
