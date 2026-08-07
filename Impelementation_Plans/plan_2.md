# healthcare+ — Final Implementation Plan (v2.0)
### Multi-Hospital Healthcare Operating System — PERN Stack

> **Status**: Architecture-approved, revision-incorporated. This document supersedes the v1.0 draft. Core vision, feature set, and phased roadmap are unchanged. This version resolves the issues identified in technical review before Phase 1 begins.

---

## 0. What Changed From v1.0 (Change Log)

| # | Area | v1.0 Issue | v2.0 Fix |
|---|---|---|---|
| 1 | Tenant middleware | Header-trusted tenant ID + raw SQL string interpolation | JWT-derived tenant context + parameterized `set_config` |
| 2 | RLS + PgBouncer | `SET LOCAL` unsafe under transaction pooling | Explicit transaction wrapper per tenant-scoped request |
| 3 | Billing schema | `billing_receipts` missing `appointment_id`, contradicts ER diagram | FK added, made nullable to support standalone lab billing |
| 4 | Queue tokens | Race condition on concurrent booking | Redis atomic counter (`INCR`) per doctor/date, reconciled to Postgres |
| 5 | Ambulance dispatch | "First accept wins" not atomic | Conditional atomic `UPDATE ... WHERE status='BROADCASTING'` |
| 6 | CDS engine | Silently replaced by LLM | Hybrid model: deterministic rules stay primary, AI is advisory only |
| 7 | NFR targets | Sub-100ms p95 conflicts with AI triage latency | Per-route SLA tiers defined |
| 8 | Doctor–hospital model | 1 doctor = 1 tenant assumed | Doctor affiliation join table added |
| 9 | Compliance | HIPAA/GDPR framing for an India-first product | DPDP Act 2023 made primary; HIPAA/GDPR kept for future international tenants |
| 10 | Security | No MFA, no refresh-token reuse detection, no upload scanning | All three added to Phase 2/7 |
| 11 | WebSockets | No room-level authorization spec | Explicit per-room auth check added |
| 12 | Booking | No idempotency on payment + token creation | Idempotency key required on `/appointments/book` |

Everything else from the original plan (vision, ER diagram shape, folder structure, RBAC matrix, notification architecture, deployment topology) is retained as-is.

---

## 1. Executive Summary

healthcare+ is being rebuilt from a single-hospital Django/SQLite prototype into a multi-tenant Healthcare Operating System on the PERN stack. This plan is the build-ready specification: every schema, middleware, and workflow below is what engineers should implement directly — not a conceptual sketch.

---

## 2. System Architecture (Unchanged Shape, Hardened Detail)

```
Cloudflare DNS/WAF → NGINX → Express Node Cluster (PM2/Docker) ⇄ React CDN
                                    |
                        +-----------+-----------+
                        |                       |
                  PostgreSQL 16              Redis Cluster
                (Primary + Replica)      (Cache + PubSub + BullMQ + Token Counters)
```

**Layered backend** (Route → Controller → Service → Repository) and **feature-based frontend** (Zustand for client state, React Query for server cache) remain as originally specified in sections 12–14 of the v1.0 draft. No changes.

---

## 3. Multi-Tenancy — Corrected Implementation

### 3.1 Tenant Context Middleware (Fixed)

**Problem in v1.0**: trusted `x-tenant-id` header directly, and interpolated it into raw SQL.

**v2.0 implementation**:

```javascript
// middleware/tenant.middleware.js
export const enforceTenant = async (req, res, next) => {
  const role = req.user?.role;
  let tenantId;

  if (role === 'super_admin') {
    // Only super_admin may switch tenant context via header,
    // and only after validating the tenant exists and is active.
    const headerTenant = req.headers['x-tenant-id'];
    if (headerTenant) {
      const tenant = await tenantRepository.findActiveById(headerTenant);
      if (!tenant) return res.status(404).json({ error: 'Tenant not found' });
      tenantId = tenant.id;
    }
  } else {
    // Every other role: tenant context comes ONLY from the JWT claim,
    // never from a client-controlled header.
    tenantId = req.user?.tenantId;
    if (!tenantId) {
      return res.status(403).json({ error: 'Tenant context missing' });
    }
  }

  req.tenantId = tenantId ?? null;
  next();
};
```

### 3.2 RLS + PgBouncer-Safe Query Wrapper

**Problem in v1.0**: `SET LOCAL` outside an explicit transaction is unsafe under PgBouncer transaction pooling.

**v2.0 implementation**:

```javascript
// repository/withTenantContext.js
export async function withTenantContext(tenantId, fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    // Parameterized — never string-interpolated
    await client.query('SELECT set_config($1, $2, true)', [
      'app.current_tenant_id',
      tenantId,
    ]);
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

// Usage in a service:
const appointments = await withTenantContext(req.tenantId, (client) =>
  appointmentRepository.findQueueForDoctor(client, doctorId, date)
);
```

Every tenant-scoped repository call must go through `withTenantContext`. This is a hard rule enforced via a lint rule / code review checklist in Phase 1 (Task 1.1).

### 3.3 Doctor Multi-Hospital Affiliation (New)

```sql
CREATE TABLE doctor_affiliations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    doctor_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    department_id UUID REFERENCES departments(id),
    is_primary BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(doctor_id, tenant_id)
);
```

`users.tenant_id` remains for staff whose home tenant is fixed (admins, receptionists, lab techs, pharmacists). For doctors specifically, application logic reads from `doctor_affiliations` rather than assuming a single tenant; `users.tenant_id` for a doctor row becomes their "home"/billing tenant only.

### 3.4 Tenant Deactivation (Fixed)

`tenants.is_active` is now the only supported deactivation path. Hard-deleting a tenant row is disabled at the application layer (no `DELETE /tenants/:id` endpoint — deactivation only). `ON DELETE SET NULL` on `users.tenant_id` is retained purely as a DB-level safety net, not as the intended deactivation mechanism.

---

## 4. Database Schema — Corrected & Extended DDL

Only the deltas from v1.0 are shown below; all other tables from the original schema (section 11) are retained unchanged.

```sql
-- Appointment status: add NO_SHOW
CREATE TYPE appointment_status AS ENUM (
    'PENDING_PAYMENT', 'CONFIRMED', 'IN_PROGRESS', 'COMPLETED', 'NO_SHOW', 'CANCELLED'
);

-- Appointments: add idempotency key for booking
ALTER TABLE appointments ADD COLUMN idempotency_key VARCHAR(255) UNIQUE;

-- Billing: fix FK relationship to match ER diagram, allow standalone lab billing
ALTER TABLE billing_receipts
    ADD COLUMN appointment_id UUID REFERENCES appointments(id) ON DELETE SET NULL,
    ADD COLUMN lab_test_id UUID REFERENCES lab_tests(id) ON DELETE SET NULL;
-- At least one of appointment_id / lab_test_id must be present (app-layer check + CHECK constraint)
ALTER TABLE billing_receipts
    ADD CONSTRAINT chk_billing_has_source
    CHECK (appointment_id IS NOT NULL OR lab_test_id IS NOT NULL);

-- Ambulance units table (referenced in queries but missing from v1.0 DDL)
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
CREATE INDEX idx_ambulance_availability ON ambulance_units(is_available) WHERE is_available = TRUE;

-- Emergency dispatches: candidate tracking to support atomic accept
ALTER TABLE emergency_dispatches
    ADD COLUMN broadcast_candidate_ids UUID[] DEFAULT '{}';

-- Consent grants (referenced in section 25 but missing from v1.0 DDL)
CREATE TABLE consent_grants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    passport_id UUID NOT NULL REFERENCES patient_passports(id) ON DELETE CASCADE,
    grantee_doctor_id UUID NOT NULL REFERENCES users(id),
    grant_type VARCHAR(50) NOT NULL, -- 'CONSULTATION' | 'EMERGENCY_OVERRIDE' | 'MANUAL'
    granted_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    related_appointment_id UUID REFERENCES appointments(id)
);
CREATE INDEX idx_consent_active ON consent_grants(passport_id, grantee_doctor_id)
    WHERE revoked_at IS NULL;

-- Audit log (referenced in section 27 but missing from v1.0 DDL)
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    action VARCHAR(100) NOT NULL,
    actor_user_id UUID REFERENCES users(id),
    target_passport_id UUID REFERENCES patient_passports(id),
    metadata JSONB DEFAULT '{}'::jsonb,
    ip_address INET,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
-- Audit logs are append-only: revoke UPDATE/DELETE at the DB role level
```

---

## 5. Queue Token Generation — Atomic Implementation (New)

**Problem in v1.0**: no defined mechanism, race condition under concurrent booking.

**v2.0 implementation**: Redis atomic counter as source of truth for allocation, reconciled into Postgres.

```javascript
// services/queue.service.js
async function allocateToken(tenantId, doctorId, date) {
  const redisKey = `token_counter:${tenantId}:${doctorId}:${date}`;
  const tokenNumber = await redis.incr(redisKey);
  await redis.expire(redisKey, 60 * 60 * 24 * 2); // 2-day TTL, past appointment date

  try {
    await withTenantContext(tenantId, (client) =>
      appointmentRepository.insertAppointment(client, {
        tenantId, doctorId, date, tokenNumber, /* ... */
      })
    );
    return tokenNumber;
  } catch (err) {
    if (isUniqueViolation(err)) {
      // Extremely rare: Redis counter desynced from DB (e.g. after a Redis flush).
      // Fall back to DB-computed next token under row lock.
      return allocateTokenFallback(tenantId, doctorId, date);
    }
    throw err;
  }
}

async function allocateTokenFallback(tenantId, doctorId, date) {
  return withTenantContext(tenantId, async (client) => {
    const { rows } = await client.query(
      `SELECT COALESCE(MAX(token_number), 0) + 1 AS next
       FROM appointments
       WHERE tenant_id = $1 AND doctor_id = $2 AND appointment_date = $3
       FOR UPDATE`,
      [tenantId, doctorId, date]
    );
    return rows[0].next;
  });
}
```

Booking endpoint requires a client-supplied `Idempotency-Key` header; the server stores and checks it against `appointments.idempotency_key` before allocating a token or charging payment, preventing double-booking from retried requests.

---

## 6. Emergency Ambulance Dispatch — Atomic Lock (New)

**Problem in v1.0**: "first accept wins" had no concurrency guarantee.

**v2.0 implementation**:

```javascript
// services/emergency.service.js
async function acceptDispatch(dispatchId, driverId) {
  const { rows } = await pool.query(
    `UPDATE emergency_dispatches
     SET status = 'ACCEPTED', assigned_driver_id = $1, updated_at = NOW()
     WHERE id = $2 AND status = 'BROADCASTING'
     RETURNING *`,
    [driverId, dispatchId]
  );

  if (rows.length === 0) {
    // Someone else already accepted, or dispatch was cancelled/timed out
    return { accepted: false, reason: 'ALREADY_ASSIGNED_OR_CLOSED' };
  }

  await ambulanceRepository.markUnavailable(driverId);
  socketService.emitToRoom(`emergency:${dispatchId}`, 'dispatch:accepted', rows[0]);
  socketService.notifyLosingCandidates(dispatchId, driverId); // tell other drivers to stand down
  return { accepted: true, dispatch: rows[0] };
}
```

**Zero-candidate fallback (new)**: if the initial PostGIS radius search returns zero ambulances, escalate to 108/102 tele-dispatch immediately rather than waiting for the 15-second driver-timeout (which assumes at least one candidate was broadcast to).

```javascript
async function triggerEmergency(patientId, location) {
  const candidates = await ambulanceRepository.findNearby(location, 10000);
  const dispatch = await emergencyRepository.create({ patientId, location, candidates });

  if (candidates.length === 0) {
    await emergencyRepository.escalateToExternalDispatch(dispatch.id); // 108/102
    return dispatch;
  }

  socketService.broadcastToDrivers(candidates, dispatch);
  scheduleTimeoutEscalation(dispatch.id, 15_000);
  return dispatch;
}
```

---

## 7. Clinical Decision Support — Hybrid Model (Fixed)

**Problem in v1.0**: implied the deterministic `CDSWarningEngine` was being replaced by an LLM.

**v2.0 model**: the two AI-adjacent systems have different safety tiers and must stay architecturally separate.

| System | Role | Engine | Failure Mode If Wrong |
|---|---|---|---|
| **CDS Warning Engine** (dosage, contraindication, pediatric/allergy checks) | **Primary, blocking** | Deterministic rule engine (ported from Django prototype, extended) | Patient harm — must never be probabilistic |
| **AI Symptom Triage** (department routing) | **Advisory, non-blocking** | LLM (Gemini/GPT-4o-mini), structured JSON output | Wrong department suggestion — recoverable, low harm |
| **AI Queue Wait-Time Predictor** | Advisory | Regression model | Wrong ETA — inconvenience, not harm |

The deterministic CDS engine remains the system of record for prescription warnings. An LLM may optionally generate a plain-language *explanation* of a triggered rule for the doctor, but it never originates or suppresses a warning itself.

---

## 8. Non-Functional Requirements — Corrected SLA Tiers

**Problem in v1.0**: single global "sub-100ms p95" target contradicted by AI-backed routes.

**v2.0**:

| Route Class | Example | p95 Target |
|---|---|---|
| Core CRUD / DB-only | `/queue/live/:doctorId`, `/passport/timeline` | < 100ms |
| Real-time WebSocket events | queue updates, GPS streaming | < 50ms |
| AI-backed | `/triage/analyze` | < 3s (with loading state + disclaimer shown immediately) |
| File upload / processing | `/lab/reports/upload` | < 5s for files up to 10MB |

Availability, RLS, encryption, and observability targets from v1.0 section 5 are retained unchanged.

---

## 9. Security — Added Controls

1. **MFA (TOTP)** mandatory for `super_admin` and `hospital_admin` roles at login; optional but encouraged for `doctor`.
2. **Refresh token reuse detection**: each refresh token is single-use; on reuse of an already-rotated token, the entire token family for that session is revoked and the user is forced to re-authenticate. Implemented via a `token_family_id` stored alongside the Redis-blacklisted token record.
3. **File upload hardening** for lab PDF uploads: MIME-type allowlist, magic-byte verification (not just extension), ClamAV (or equivalent) scan before storage, 10MB size cap, storage in a non-executable S3 path with signed-URL retrieval only.
4. **WebSocket room authorization**: on `socket.io` connection, the server validates the JWT and, for every `join(room)` call, checks that the authenticated user is entitled to that room (e.g., a patient may only join `queue:{doctorId}:{date}` rooms tied to their own active appointment; a patient may only join `emergency:{dispatchId}` for their own dispatch).

```javascript
socket.on('join', async ({ room }) => {
  const authorized = await authorizeRoomAccess(socket.user, room);
  if (!authorized) return socket.emit('error', 'Unauthorized room access');
  socket.join(room);
});
```

5. **Receipt signing key rotation**: move from a single global HMAC secret to per-tenant derived signing keys (HKDF from a platform master key + `tenant_id`), so a single key compromise doesn't invalidate every receipt platform-wide.

---

## 10. Compliance — India-First Framing (Corrected)

**Problem in v1.0**: cited HIPAA/GDPR as primary compliance targets for a product priced in ₹ and integrated with India's 108/102 emergency numbers.

**v2.0**:
- **Primary**: India's **Digital Personal Data Protection (DPDP) Act, 2023** — consent-first data collection, data principal rights (access/correction/erasure), breach notification requirements, data localization considerations for health data.
- **Retention vs. erasure conflict**: medical record retention norms (commonly multi-year under Indian clinical establishment rules) take precedence over a raw "delete on request" pattern. Erasure requests are implemented as **anonymization** (strip direct identifiers, retain de-identified clinical record) rather than hard deletion, with this policy stated explicitly in the patient consent flow.
- **Secondary / future**: HIPAA and GDPR alignment retained as a roadmap item for when the platform onboards US/EU tenants — architected for, not fully implemented, in v1.
- **ABDM/FHIR** (previously listed under "Future Enhancements") is pulled forward: the Healthcare Passport ID field should be structured to accept an ABHA (Ayushman Bharat Health Account) number from day one, even if the integration itself ships later. Retrofitting this field into a live `patient_passports` table is far more expensive than including it now.

```sql
ALTER TABLE patient_passports ADD COLUMN abha_id VARCHAR(50) UNIQUE;
```

---

## 11. Notifications — Added Consent Ledger

Per TRAI regulations on commercial/transactional SMS consent:

```sql
CREATE TABLE notification_consents (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id),
    channel VARCHAR(20) NOT NULL, -- SMS | EMAIL | PUSH
    consented BOOLEAN NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

BullMQ notification workers check this table before dispatching to SMS/Email channels; in-app and push notifications for critical events (appointment, emergency) remain unaffected by consent status.

---

## 12. Revised Development Roadmap

```
Phase 1: Foundations (Weeks 1-2)
   + Tenant middleware + withTenantContext wrapper (Section 3.1-3.2) as a Phase 1 deliverable, not deferred

Phase 2: Auth & Healthcare Passport (Weeks 3-4)
   + MFA for admin roles
   + Refresh token reuse detection
   + ABHA ID field on passport schema

Phase 3: Multi-Hospital & Doctor Console (Weeks 5-6)
   + doctor_affiliations table and affiliation-aware queries
   + Redis-based atomic token allocation

Phase 4: AI Triage & Live Queue Engine (Weeks 7-8)
   + Hybrid CDS model (deterministic engine ported first, AI explanation layer second)
   + Per-route SLA tiers enforced in load tests

Phase 5: Real-Time Emergency SOS & Ambulance System (Weeks 9-10)
   + Atomic accept-lock on emergency_dispatches
   + Zero-candidate immediate escalation path
   + WebSocket room authorization

Phase 6: Lab, Pharmacy & Billing Integration (Weeks 11-12)
   + billing_receipts FK fix (appointment_id / lab_test_id)
   + File upload hardening (AV scan, magic-byte check)
   + Per-tenant receipt signing keys

Phase 7: Security Audit, Testing & Production Deployment (Weeks 13-14)
   + Cross-tenant RLS penetration test (explicit test suite, not just RLS policy existence)
   + PgBouncer transaction-pooling load test to confirm tenant context isolation under concurrency
   + DPDP compliance checklist sign-off
```

---

## 13. Updated Production Readiness Checklist

- [ ] All PostgreSQL tables created with foreign keys, indexes, RLS policies **and passing an automated cross-tenant isolation test suite**
- [ ] Tenant context resolved exclusively from JWT for non-super-admin roles; header-based override restricted to super_admin with tenant-active validation
- [ ] All tenant-scoped queries routed through `withTenantContext` (enforced via lint rule)
- [ ] Redis token counters + Postgres fallback path tested under simulated concurrent booking load
- [ ] Ambulance accept-lock tested under simulated concurrent driver acceptance
- [ ] Deterministic CDS engine ported and unit-tested against known contraindication/dosage cases before AI triage ships
- [ ] MFA enforced for hospital_admin and super_admin
- [ ] Refresh token rotation + reuse detection verified
- [ ] Lab upload pipeline scanning + type validation verified
- [ ] WebSocket room authorization tested (patient cannot join another patient's queue/emergency room)
- [ ] DPDP consent flows and anonymization-on-erasure policy documented and implemented
- [ ] Per-tenant receipt signing key rotation mechanism in place
- [ ] Redis eviction/persistence configured
- [ ] JWT secrets and DB credentials in Secrets Manager
- [ ] Rate limiting on auth and emergency endpoints
- [ ] CORS locked to trusted domains
- [ ] TLS certificates with auto-renewal
- [ ] Hourly WAL archiving + daily full DB backup verified with a tested restore drill

---

## 14. Sections Retained Unchanged From v1.0

The following sections of the original document require no changes and should be used as-is: Product Vision (§3), Functional Requirements (§4), Wireframes (§7), Information Architecture (§8), Frontend Architecture (§13), Folder Structures (§14), API Endpoint List (§15, extend with idempotency header note), RBAC Matrix (§17), Notification Architecture (§18, extended per §11 above), Real-Time Room Structure (§20), Deployment Topology (§28), Testing Strategy (§29), Monitoring & Observability (§30), Risks & Mitigation (§35, all three original risks remain valid), Future Enhancements (§36, minus ABDM/FHIR which is now pulled into §10 above).

---

*End of v2.0. This document is the authoritative build reference — where it conflicts with v1.0, v2.0 governs.*
