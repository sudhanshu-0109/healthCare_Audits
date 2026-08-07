# healthcare+ — ULTIMATE MASTER IMPLEMENTATION BLUEPRINT (Consolidated Final)

> **Document status:** Single source of truth. Supersedes and merges all seven prior planning documents (`plan_1.md` v1.0, `plan_2.md` v2.0, `plan_Final_last_.md` v2.0-Final, `implementation_plan0000001.md` v4.0, `implementation_plan_Finallllll.md` v5.0 mobile-first, `healthcare-plus-master-plan-v3_1.md` v3.1, `frontend_implementation_plan.md` standalone frontend v1.0).
> **Target stack:** PostgreSQL 16+ (PostGIS) · Express.js / Node.js 20+ LTS · React 18+ (Vite) · Redis 7+ · Socket.io · BullMQ · GSAP 3 + ScrollTrigger.
> **Product model:** Multi-hospital SaaS "Healthcare Operating System" with a Universal Patient Healthcare Passport.
> **Compliance target:** India DPDP Act 2023 (primary), TRAI SMS/consent regulations, ABDM ABHA standard. HIPAA/GDPR retained as a future-international roadmap item, not a v1 requirement.
> **How to use this document:** Start at Phase 0 and proceed sequentially. Each phase is self-contained (objective → features → tasks → folders → components → APIs → DB → UI/UX → validation → dependencies → outcome → completion criteria) and unblocks the next. Nothing here should require another planning document — if you hit a genuine gap while building, it is a bug in this document, not a reason to re-plan.

---

## 0. Document Provenance & Merge Resolution Notes

The seven source documents are five iterations of the same backend/architecture spec (v1.0 → v2.0 → v2.0-Final → v4.0 → v5.0) plus one always-current schema branch (v3.1) plus one standalone frontend design spec. They agree on ~90% of substance; where they disagree, this section states the resolution so nobody has to reverse-engineer *why* a decision was made.

### 0.1 What each source contributed

| Source | Unique contribution kept in this document |
|---|---|
| `plan_1.md` (v1.0) | Original codebase audit of the Django/SQLite prototype (§Phase 0 rationale), full RBAC matrix, folder structures, notification architecture (BullMQ fan-out), risks register, testing/monitoring strategy, future enhancements list. |
| `plan_2.md` (v2.0) | The critical security corrections: JWT-only tenant context, parameterized `set_config`, `withTenantContext` wrapper, atomic Redis token counter, atomic ambulance accept lock, hybrid CDS model, DPDP-first compliance framing, tiered SLAs. |
| `plan_Final_last_.md` (v2.0-Final) | Consolidation of v2.0's fixes into clean, final-form code; break-glass audit log framing; per-tenant receipt signing key rationale. |
| `implementation_plan0000001.md` (v4.0) | Architectural comparison matrix (legacy → final), MoSCoW feature prioritization, full API specification matrix with SLA tiers, role-based state machines, wireframe ASCII layouts. |
| `implementation_plan_Finallllll.md` (v5.0) | Mobile-first device-tier responsive strategy, GSAP/ScrollTrigger animation engine, dual dark/light token system, week-numbered roadmap. |
| `healthcare-plus-master-plan-v3_1.md` (v3.1) | The most bug-patched schema and backend: `tenant_settings`, `departments` as a real FK'd table, `dose_logs`, in-app `notifications` table, AES-256-GCM MFA secret encryption, Lite Appointment Redis lock with 0.1-increment retry, doctor–tenant affiliation trigger, idempotency middleware, full edge-case/failure-mode matrix, RLS on all 11 tenant-scoped tables (not just 2). |
| `frontend_implementation_plan.md` (standalone) | The definitive, most detailed frontend spec: page-by-page UI map (14 screens), organic/anti-boxy design principles, GSAP utility functions, phased frontend roadmap (F0–F6). |

### 0.2 Conflicts identified and how they are resolved

| # | Conflict | Resolution used in this document | Why |
|---|---|---|---|
| 1 | `implementation_plan_Finallllll.md` specifies a **dark-mode-default** token system with a `.light-mode` override class. `frontend_implementation_plan.md` mandates **light theme only, zero dark mode**, with a materially different (and more polished) palette application. | **Light theme only.** Dark mode is explicitly dropped from scope. | The standalone frontend document is the more recent, more deliberate, single-purpose design spec — it was written specifically to fix "boxy AI-generated" visual patterns, which the dark-mode version does not address. Maintaining two theme systems doubles frontend QA surface for no product requirement anyone asked for. If dark mode is wanted later, it belongs in §18 Future Enhancements, not v1. |
| 2 | Backend DDL is duplicated near-verbatim across `v3.1`, `v4.0`, and `v5.0`, with `v3.1` being the most complete (has `tenant_settings`, real `departments` table, `dose_logs`, `notifications`, RLS on 11 tables; the others RLS only 10 and are missing `tenant_settings` in the RLS loop). | **v3.1's schema is the canonical schema**, reproduced in full in §4, with v5.0's `primary_color` default and v4.0's comparison-matrix framing merged in as commentary. | v3.1 is the only version whose own change-log explicitly closes every gap the others left open (fallback token function undefined, room-auth function undefined, MFA stored in plaintext, doctor–tenant integrity unenforced). Regressing to an earlier draft's schema would silently reintroduce fixed bugs. |
| 3 | `plan_1.md`/`plan_2.md` use `mfa_secret VARCHAR(255)` (plaintext); `v3.1` uses `mfa_secret_encrypted` + `mfa_secret_iv` with AES-256-GCM. | **Encrypted storage (v3.1 pattern).** | Storing a TOTP seed in plaintext is a critical vulnerability — anyone with read access to the `users` table could generate valid MFA codes for any admin account. Non-negotiable fix. |
| 4 | Early drafts type `token_number` as `INT`; from v2.0 onward it's `NUMERIC(6,1)` to support Lite Appointments (`15.5`). | **`NUMERIC(6,1)`**, consolidated from v2.0 onward. | Required for the Lite Appointment feature, which is in scope from Phase 3 onward in every later draft. |
| 5 | `doctor_affiliations` in `plan_Final_last_.md` uses a free-text `specialty VARCHAR(100)`; from v2.0-DDL-delta onward it's `department_id UUID REFERENCES departments(id)`. | **FK'd `department_id`**, with a proper `departments` table (v3.1). | Free-text specialty can't be joined against a hospital's actual department list for the "AI suggests a department this hospital doesn't have" dead-end problem described in v3.1 §2.1 — this is a real UX bug the FK model prevents structurally. |
| 6 | `billing_receipts` schema disagrees across drafts: v1.0 has flat `consultation_fee`/`lab_fee`/`pharmacy_fee` columns; v2.0 adds `lab_test_id`; v3.1/v4.0/v5.0 replace both with a generic `appointment_id` / `lab_request_id` / `pharmacy_order_id` triple plus a normalized `receipt_items` line-item table. | **Normalized `receipt_items` model (v3.1/v4.0/v5.0 pattern).** | Flat fee columns can't represent "two lab tests + one pharmacy order on one receipt," which the product vision (§1) explicitly requires ("every charge... stored as individual immutable line items," per `plan_1.md` §23 itself). The normalized model is what the later drafts converged on for good reason. |
| 7 | Ambulance emergency-status enum: early drafts stop at `ARRIVED_HOSPITAL`/`CANCELLED`; v3.1 onward adds `ESCALATED_108` as a first-class terminal state. | **Include `ESCALATED_108`.** | Zero-candidate and driver-timeout escalation paths (§Phase 5) need a terminal status to land on; without it the dispatch row has no valid state to represent "no ambulance came, we called 108 instead." |
| 8 | Consent table is named `consent_grants` in v1.0/v2.0/v2.0-Final and `passport_consent_grants` in v3.1/v4.0/v5.0, with a `tenant_id` column added in v3.1 for RLS scoping. | **`passport_consent_grants`, with `tenant_id`.** | RLS cannot scope a table with no tenant column; this is a structural requirement, not a naming preference. |
| 9 | Frontend roadmap week numbers differ between v4.0 (Phase 6–7, weeks 11–14), v5.0 (Phase F0 + Phase 1–7, weeks 1–15), and the standalone frontend plan (Phase F0–F6, weeks 1–12, frontend-only timeline). | **Frontend and backend run as two coordinated tracks inside one 16-week program** (§13), reconciled so frontend work never starts building a screen before the API it depends on exists. See the phase-by-phase dependency notes. | Running frontend as a fully separate 12-week timeline (as the standalone doc implies in isolation) would have engineers building the Doctor Desk CDS banner before the Hybrid CDS Engine endpoint exists. Interleaving fixes this without dropping any task from either roadmap. |

Nothing in this resolution list removes a feature, weakens a security control, or changes the product vision — every resolution is either "take the more complete/secure version" or "make two independently-correct specs execute in a compatible order."

### 0.3 What this document adds beyond the source material

Acting as reviewing architect/PM/tech lead, the following gaps in the merged source material are filled in explicitly rather than left implicit (flagged inline with **[NEW]** where they first appear):

- A concrete **state-management strategy** section separating Zustand-owned vs. React-Query-owned vs. Socket.io-owned state, since no source document draws this line explicitly beyond a diagram.
- A concrete **API integration order** for the frontend, since building screens before their backing endpoints exist is the single most common cause of wasted rework.
- An **accessibility requirements** section (WCAG 2.1 AA baseline) — referenced nowhere in any of the seven source documents despite being a hard requirement for a healthcare product.
- An explicit **error-handling contract** (HTTP error shape, frontend error boundary strategy, offline/retry behavior for the SOS flow specifically, since a failed emergency request is not an acceptable silent failure).
- A **environment/config management** plan (`.env` schema, secrets rotation) since none of the source docs specify what actually goes in `environment.js`.
- A **database migration workflow** (the source docs specify final-state DDL but never how you get from empty database to that DDL safely in a team setting).
- A **data seeding / demo-data plan** for local development, since a multi-tenant spatial app is unusable to develop against without seeded tenants, doctors, and geocoded addresses.
- Explicit **payment gateway integration points**, referenced constantly ("Patient pays," "Payment Gateway") but never specified as an actual integration in any source document.

---

## 1. Executive Summary & Product Vision

**healthcare+** is being rebuilt from a single-hospital Django/SQLite prototype into a multi-tenant Healthcare Operating System on the PERN stack (PostgreSQL, Express, React, Node). Hospitals subscribe as tenants to host their digital workflows — queues, EHR, billing, labs, pharmacy, emergency response — while patients use one unified web/mobile application backed by a cross-hospital **Universal Healthcare Passport**.

```
+-----------------------------------------------------------------------------------+
|                                  healthcare+                                      |
|                       Multi-Hospital Healthcare Network                           |
+-----------------------------------------------------------------------------------+
                                         |
     +-----------------------------------+-----------------------------------+
     |                                                                       |
+----+----------------------------------+       +----------------------------+----+
|        Central Healthcare Passport    |       |       Hospital Tenant A        |
|  - Universal Medical History          |       | - Isolated Operational Data    |
|  - Cross-Hospital Consent Matrix      |       | - Custom Queue & Doctors       |
|  - Immunizations & Chronic Conditions |       | - Hospital Labs & Pharmacy     |
+---------------------------------------+       +---------------------------------+
                                         |
                                                +----------------------------+----+
                                                |       Hospital Tenant B        |
                                                | - Isolated Operational Data    |
                                                | - Custom Queue & Doctors       |
                                                | - Hospital Labs & Pharmacy     |
                                                +---------------------------------+
```

### 1.1 Core pillars (why each exists)

1. **Multi-tenancy with real isolation** — a hospital's queue, billing, and staff data must be provably inaccessible to any other hospital, enforced at the database layer (RLS), not just the application layer, because application-layer-only isolation has historically been the #1 cause of real-world multi-tenant SaaS data leaks.
2. **Universal Healthcare Passport** — the patient, not any single hospital, owns their longitudinal medical record; hospitals get *consented, time-boxed* access, not permanent access, because that's both a DPDP compliance requirement and the actual value proposition of a *network* product over N independent hospital apps.
3. **Live queue with real tokens** — replaces "come back at 4pm and hope" with a numeric position and ETA, which is the single feature patients in this market complain about most in any Indian OPD.
4. **Emergency SOS with a real fallback** — an ambulance dispatch system that silently fails when zero drivers are nearby is worse than no feature at all; the 108/102 tele-dispatch escalation path is a first-class part of the design, not an edge case bolted on later.
5. **Hybrid, not purely AI, safety layer** — anything that can cause patient harm (drug contraindication, dosage, allergy) is decided by a deterministic rule engine; AI is confined to *advisory* roles (department routing, plain-language explanations) where being wrong is inconvenient, not dangerous.

### 1.2 In scope vs. explicitly out of scope for v1

**In scope (Must/Should, §3):** everything in Phases 0–8 below.
**Explicitly out of scope for v1 (Future, §18):** WebRTC telemedicine video, Bluetooth LE IoT vitals streaming, full ABDM/FHIR interoperability (the `abha_id` *field* ships in v1; the *integration* does not), dark mode, multi-currency/international billing, HIPAA/GDPR full compliance program.

---

## 2. Architecture Overview

### 2.1 Infrastructure topology

```
                                  [ Cloudflare WAF / DNS ]
                                             |
                                  [ NGINX Reverse Proxy — SSL/TLS termination ]
                                             |
                  +--------------------------+--------------------------+
                  v                                                     v
      [ Express Node Cluster (PM2, N instances) ]               [ React Frontend CDN ]
                  |                                              (Vite static build)
        +---------+---------+
        v                   v
[ PostgreSQL 16 Primary ]   [ Redis 7+ Cluster ]
(PostGIS + Read Replica)    (Cache + Pub/Sub + BullMQ + Token/Slot Locks)
        ^
   [ PgBouncer — Transaction Mode ]
```

- **Cloudflare** — DNS, WAF, DDoS mitigation, edge caching for static frontend assets.
- **NGINX** — TLS termination, reverse proxy to the Node cluster, gzip/brotli compression, WebSocket upgrade passthrough for Socket.io.
- **Express/Node cluster** — stateless, PM2-managed, horizontally scalable; all session/queue/lock state lives in Redis or Postgres, never in process memory, so any instance can serve any request.
- **PostgreSQL 16 + PostGIS** — system of record; PostGIS provides `ST_DWithin`/`ST_Distance` spatial queries for hospital discovery and ambulance dispatch.
- **PgBouncer (transaction mode)** — connection pooling for thousands of concurrent tenant connections; **this has a direct implication for RLS** (§4.3) that every engineer on the project must understand before writing a single query.
- **Redis 7+** — atomic token counters, Lite Appointment locks, Socket.io Pub/Sub adapter backplane, BullMQ job queue, refresh-token family blacklist, 60s crowd-status cache.
- **React 18 + Vite CDN** — statically built, served from edge CDN, talks to the API over HTTPS and to Socket.io over WSS.

### 2.2 Technology stack justification

| Choice | Selected over | Why |
|---|---|---|
| PostgreSQL 16 + PostGIS | MySQL / MongoDB | Native spatial queries (`ST_DWithin`, `ST_Distance`), JSONB for flexible vitals/allergy data, mature Row-Level Security for multi-tenant isolation, strong transactional integrity for billing/queue correctness. |
| Express.js / Node 20+ LTS | Django/Python | High-throughput non-blocking I/O for WebSocket-heavy real-time features (live queue, GPS tracking), unified JS/TS tooling across stack, mature queue-processing ecosystem (BullMQ). |
| React 18 + Vite | Next.js / Vue | Component-driven UI at speed; Vite's sub-second HMR matters for a 14-screen, multi-role frontend; no SSR requirement here (patient app is behind auth), so Next's main advantage doesn't apply. |
| Zustand + TanStack Query v5 | Redux / plain Context | Zustand for small, fast client-only state (auth session, SOS local state, UI toggles) without boilerplate; React Query owns all server-derived state (hospitals, appointments, passport timeline) with automatic caching/invalidation/retry — see §7.3 for the explicit ownership split. |
| PgBouncer + Redis | Direct pool-per-instance | PgBouncer keeps Postgres connection counts sane under thousands of tenant connections; Redis serves four independent jobs (counter, Pub/Sub, queue, cache) that would otherwise require four separate pieces of infrastructure. |
| Socket.io | Raw WebSockets | Built-in room model (used heavily — `queue:{tenantId}:{doctorId}`, `emergency:{dispatchId}`), automatic reconnection/fallback, and a Redis adapter for multi-instance Pub/Sub out of the box. |
| GSAP 3 + ScrollTrigger | CSS transitions / Framer Motion | Fine-grained control needed for the SOS pulse and scroll-reveal patterns across 14 screens without fighting CSS specificity; GSAP's timeline API keeps animation code centralized in `utils/gsapAnimations.js` rather than scattered per-component. |

### 2.3 Architectural comparison — legacy vs. this blueprint

| Dimension | Legacy Django/SQLite prototype | This blueprint (Final) |
|---|---|---|
| Multi-tenancy | Single-hospital monolith, no tenant boundary | RLS enforced on all 11 tenant-scoped tables + emergency-dispatch tenant policy; JWT-only tenant context; `withTenantContext` PgBouncer-safe wrapper |
| Database | SQLite | PostgreSQL 16 + PostGIS, 15 tables, GIST spatial indexes, DB-level integrity triggers |
| Token queue engine | Sequential in-memory loop | Atomic Redis `INCR` + Postgres `FOR UPDATE` fallback + Redis-locked fractional Lite tokens |
| Emergency dispatch | Static string location, FCFS | PostGIS 10km spatial search, atomic conditional `UPDATE...RETURNING` lock, immediate zero-candidate 108/102 escalation |
| CDS | Regex string matching | Hybrid: deterministic blocking gate + LLM advisory explanation layer |
| Security | Plaintext passwords, mock JWTs | Argon2id, RS256 JWT rotation + refresh-family revocation, AES-256-GCM MFA secrets, Ed25519 receipt signatures, ClamAV upload scanning |
| Real-time | Manual polling | Authenticated Socket.io, JWT handshake, per-room authorization guard |
| Design | Basic CSS, boxy defaults | Organic light-theme design system, GSAP ScrollTrigger, mobile-first device-tier layouts |

---

## 3. MoSCoW Feature Prioritization

```
+-----------------------------------------------------------------------------------+
|  MUST HAVE (Phases 1-5 core — nothing ships without these)                        |
|  - PostgreSQL RLS multi-tenancy & parameterized tenant context middleware          |
|  - Argon2id auth, RS256 JWT rotation, refresh-token family revocation, TOTP MFA    |
|  - Universal Healthcare Passport, ABHA ID field, DPDP 2023 soft anonymization      |
|  - Doctor multi-hospital affiliations & department schema                         |
|  - Idempotent appointment booking & atomic Redis queue token allocation           |
|  - PostGIS emergency SOS, atomic driver lock, 108/102 tele-dispatch fallback      |
|  - Hybrid CDS engine (deterministic core gate + LLM advisory summaries)           |
|  - Pharmacy order fulfillment & laboratory PDF report upload (ClamAV-scanned)     |
|  - Itemized billing & Ed25519 cryptographic receipt signatures                    |
+-----------------------------------------------------------------------------------+
|  SHOULD HAVE (Phase 5-6 enhancements)                                             |
|  - Lite Appointments (fractional token, e.g. #15.5)                               |
|  - AI Crowd Status computation engine (🟢 Low / 🟡 Moderate / 🔴 High)             |
|  - Automated medicine reminders & dose logs (`dose_logs`)                         |
|  - AI symptom triage banner (natural language → department mapping)              |
|  - Persistent in-app notifications feed (`notifications` table)                   |
+-----------------------------------------------------------------------------------+
|  COULD HAVE (Phase 7 polish)                                                      |
|  - Advanced hospital explorer search & filter grid                                |
|  - Doctor rating & patient feedback module                                        |
|  - Manual queue token override audit trail (in scope as MUST for admin safety —   |
|    see §4 `audit_logs`; the *UI polish* around it is COULD)                       |
+-----------------------------------------------------------------------------------+
|  FUTURE SCOPE (post-launch, §18)                                                  |
|  - WebRTC peer-to-peer telemedicine video                                         |
|  - Bluetooth LE IoT vitals streaming                                              |
|  - Full ABDM FHIR health-data-exchange interoperability                           |
|  - Dark mode theme                                                                |
+-----------------------------------------------------------------------------------+
```


---

## 4. Complete Database Design (Canonical DDL — v3.1-derived, Final)

### 4.1 Design principles

- Every operational (tenant-owned) table carries `tenant_id` and is covered by an RLS policy — no exceptions, no "we'll add it later" tables.
- All primary keys are `UUID` (`uuid_generate_v4()`) — never auto-increment integers, since tenant/patient IDs must not leak sequential information (e.g. total patient count) and must be safely generatable client-side for idempotency keys.
- Money fields are `NUMERIC(10,2)`, never `FLOAT`/`REAL` — floating point billing math is a bug class this project does not get to have.
- Every status field is a Postgres `ENUM`, not a free-text `VARCHAR`, so invalid states are rejected by the database itself, not just application validation.
- `token_number` is `NUMERIC(6,1)` specifically to support Lite Appointment fractional tokens (`15.5`).
- Soft-deletion (`is_active`) is preferred over hard deletion everywhere a row has referential history (tenants, users, doctor affiliations, departments) — hard `DELETE` endpoints are not built for these entities.

### 4.2 Full DDL (run in order; this is the single migration source of truth)

```sql
-- ============================================================
-- EXTENSIONS
-- ============================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";

-- ============================================================
-- 1. TENANTS (Hospital SaaS tenants)
-- ============================================================
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) UNIQUE NOT NULL,
    address TEXT NOT NULL,
    location GEOMETRY(Point, 4326),
    contact_email VARCHAR(255) NOT NULL,
    contact_phone VARCHAR(50) NOT NULL,
    signing_public_key TEXT NOT NULL,      -- Ed25519 public key, digital receipt verification
    is_active BOOLEAN DEFAULT TRUE,        -- ONLY supported deactivation path — no hard DELETE endpoint
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_tenants_location ON tenants USING GIST(location);

-- 1a. TENANT SETTINGS — branding, hours, capacity (feeds AI Crowd Status engine)
CREATE TABLE tenant_settings (
    tenant_id UUID PRIMARY KEY REFERENCES tenants(id) ON DELETE CASCADE,
    logo_url TEXT,
    primary_color VARCHAR(20) DEFAULT '#03A6A1',
    working_hours JSONB DEFAULT '{}'::jsonb,           -- {"mon": ["09:00","18:00"], ...}
    daily_token_capacity_per_doctor INT DEFAULT 40,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- 2. USERS & ROLES
-- ============================================================
CREATE TYPE user_role AS ENUM (
    'super_admin', 'hospital_admin', 'doctor', 'receptionist',
    'lab_tech', 'pharmacist', 'nurse', 'patient', 'ambulance_driver', 'support'
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE RESTRICT,  -- NULL for patients & super_admins
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,               -- Argon2id
    full_name VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    role user_role NOT NULL,
    mfa_secret_encrypted TEXT,                          -- AES-256-GCM ciphertext, NEVER plaintext
    mfa_secret_iv VARCHAR(32),                          -- initialization vector for decryption
    is_mfa_enabled BOOLEAN DEFAULT FALSE,
    is_verified BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_users_tenant_role ON users(tenant_id, role);

-- ============================================================
-- 3. DEPARTMENTS (FK'd, not free text — see §0.2 conflict #5)
-- ============================================================
CREATE TABLE departments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, name)
);

-- ============================================================
-- 4. DOCTOR MULTI-HOSPITAL AFFILIATIONS
-- ============================================================
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

-- ============================================================
-- 5. HEALTHCARE PASSPORTS (universal identity, DPDP compliance)
-- ============================================================
CREATE TABLE patient_passports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    patient_id UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    abha_id VARCHAR(50) UNIQUE,                         -- ABDM National Health ID (field ships now, integration later)
    blood_group VARCHAR(10),
    allergies JSONB DEFAULT '[]'::jsonb,
    chronic_conditions JSONB DEFAULT '[]'::jsonb,
    emergency_contact JSONB NOT NULL,
    is_anonymized BOOLEAN DEFAULT FALSE,                -- DPDP Act 2023 right-to-erasure (soft, not hard delete)
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- 6. PASSPORT CONSENT GRANTS
-- ============================================================
CREATE TABLE passport_consent_grants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    passport_id UUID NOT NULL REFERENCES patient_passports(id) ON DELETE CASCADE,
    grantee_doctor_id UUID NOT NULL REFERENCES users(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),     -- required for RLS scoping
    grant_type VARCHAR(50) NOT NULL,                    -- 'APPOINTMENT_AUTO' | 'EMERGENCY_OVERRIDE' | 'MANUAL_GRANT'
    granted_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    related_appointment_id UUID
);
CREATE INDEX idx_consent_active ON passport_consent_grants(passport_id, grantee_doctor_id) WHERE revoked_at IS NULL;

-- ============================================================
-- 7. APPOINTMENTS & QUEUE (regular + fractional Lite tokens)
-- ============================================================
CREATE TYPE appointment_type AS ENUM ('REGULAR', 'LITE_FOLLOWUP');
CREATE TYPE appointment_status AS ENUM ('PENDING_PAYMENT', 'CONFIRMED', 'IN_PROGRESS', 'COMPLETED', 'NO_SHOW', 'CANCELLED');

CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE RESTRICT,
    patient_id UUID NOT NULL REFERENCES users(id),
    doctor_id UUID NOT NULL REFERENCES users(id),
    appointment_date DATE NOT NULL,
    slot_time TIME NOT NULL,
    token_number NUMERIC(6,1) NOT NULL,                 -- fractional to support e.g. 15.5
    type appointment_type DEFAULT 'REGULAR',
    status appointment_status DEFAULT 'PENDING_PAYMENT',
    consultation_fee NUMERIC(10,2) NOT NULL,
    idempotency_key VARCHAR(255) UNIQUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, doctor_id, appointment_date, token_number)
);
CREATE INDEX idx_appointments_queue ON appointments(tenant_id, doctor_id, appointment_date, status);

-- DB-level guarantee: cannot book a doctor at a hospital they aren't affiliated with
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

-- ============================================================
-- 8. PRESCRIPTIONS & MEDICINES
-- ============================================================
CREATE TABLE prescriptions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    appointment_id UUID REFERENCES appointments(id),
    patient_id UUID NOT NULL REFERENCES users(id),
    doctor_id UUID NOT NULL REFERENCES users(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    diagnosis TEXT NOT NULL,
    instructions TEXT,
    cds_evaluation JSONB,                               -- Hybrid CDS Engine result, audit trail
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE prescription_medicines (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    prescription_id UUID NOT NULL REFERENCES prescriptions(id) ON DELETE CASCADE,
    medicine_name VARCHAR(255) NOT NULL,
    dosage_per_day VARCHAR(50) NOT NULL,                -- e.g. "1-0-1", "1-1-1-1", "SOS"
    duration_days INT NOT NULL,
    special_instructions VARCHAR(255)
);

-- ============================================================
-- 9. PHARMACY ORDERS
-- ============================================================
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

-- 9a. DOSE LOGS — medicine reminder schedule (drives dashboard checkboxes)
CREATE TABLE dose_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    prescription_medicine_id UUID NOT NULL REFERENCES prescription_medicines(id) ON DELETE CASCADE,
    patient_id UUID NOT NULL REFERENCES users(id),
    scheduled_for TIMESTAMPTZ NOT NULL,
    taken_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_dose_logs_pending ON dose_logs(patient_id, scheduled_for) WHERE taken_at IS NULL;

-- ============================================================
-- 10. LABORATORY REQUESTS
-- ============================================================
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

-- ============================================================
-- 11. BILLING (normalized — see §0.2 conflict #6)
-- ============================================================
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
    digital_signature TEXT NOT NULL,                    -- Ed25519 signature, verifiable via tenant public key
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_billing_has_source CHECK (
      appointment_id IS NOT NULL OR lab_request_id IS NOT NULL OR pharmacy_order_id IS NOT NULL
    )
);

CREATE TABLE receipt_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    receipt_id UUID NOT NULL REFERENCES billing_receipts(id) ON DELETE CASCADE,
    item_type VARCHAR(50) NOT NULL,                     -- 'CONSULTATION' | 'LAB_TEST' | 'MEDICINE' | 'TAX' | 'DISCOUNT'
    description VARCHAR(255) NOT NULL,
    quantity INT NOT NULL DEFAULT 1,
    unit_price NUMERIC(10,2) NOT NULL,
    total_price NUMERIC(10,2) NOT NULL
);

-- ============================================================
-- 12. AMBULANCE FLEET & EMERGENCY DISPATCH
-- ============================================================
CREATE TABLE ambulance_units (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id),
    driver_id UUID NOT NULL REFERENCES users(id),
    vehicle_number VARCHAR(50) NOT NULL,
    current_location GEOMETRY(Point, 4326),
    is_available BOOLEAN DEFAULT TRUE,                  -- driver's Online/Offline toggle
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_ambulance_location ON ambulance_units USING GIST(current_location);
CREATE INDEX idx_ambulance_availability ON ambulance_units(is_available) WHERE is_available = TRUE;

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

-- ============================================================
-- 13. AUDIT & CONSENT LEDGER
-- ============================================================
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    action VARCHAR(100) NOT NULL,
    actor_user_id UUID REFERENCES users(id),
    target_passport_id UUID REFERENCES patient_passports(id),
    metadata JSONB DEFAULT '{}'::jsonb,
    ip_address INET,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
-- Append-only: application DB role gets INSERT/SELECT only, no UPDATE/DELETE grant (enforced in Phase 1 Task 1.4)

CREATE TABLE notification_consents (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id),
    channel VARCHAR(20) NOT NULL,                       -- 'SMS' | 'EMAIL' | 'PUSH'
    consented BOOLEAN NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 14. IN-APP NOTIFICATIONS (persisted bell/inbox feed, distinct from SMS/email dispatch)
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id),
    tenant_id UUID REFERENCES tenants(id),
    category VARCHAR(50) NOT NULL,                      -- 'APPOINTMENT' | 'LAB' | 'PHARMACY' | 'BILLING' | 'QUEUE' | 'EMERGENCY'
    title VARCHAR(255) NOT NULL,
    body TEXT,
    is_read BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_notifications_unread ON notifications(user_id, is_read) WHERE is_read = FALSE;

-- ============================================================
-- ROW LEVEL SECURITY — applied to every tenant-scoped table
-- ============================================================
DO $$
DECLARE
  t TEXT;
BEGIN
  FOREACH t IN ARRAY ARRAY[
    'users', 'departments', 'doctor_affiliations', 'appointments',
    'prescriptions', 'pharmacy_orders', 'lab_requests', 'billing_receipts',
    'ambulance_units', 'passport_consent_grants', 'tenant_settings'
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

-- emergency_dispatches uses assigned_tenant_id, nullable pre-acceptance (still broadcasting, unassigned)
ALTER TABLE emergency_dispatches ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_emergency_dispatches ON emergency_dispatches
    FOR ALL USING (
      assigned_tenant_id = NULLIF(current_setting('app.current_tenant_id', true), '')::uuid
      OR assigned_tenant_id IS NULL
    );
```

> **Migration note:** `users` has RLS enabled. Auth routes (`/login`, `/register`) run **before** tenant context exists, so the auth service must connect using a Postgres role that bypasses RLS (`BYPASSRLS`) strictly for the login/lookup-by-email path, or perform that lookup via a `SECURITY DEFINER` function. Document this exception prominently in the Phase 2 auth module README — it is the one legitimate RLS exception in the schema and a common source of confusion for new engineers on the project.

### 4.3 Why RLS + PgBouncer needs a wrapper, not just `SET LOCAL`

Under PgBouncer's **transaction pooling** mode, a bare `SET LOCAL app.current_tenant_id = ...` executed outside an explicit transaction is unsafe: the connection can be handed back to the pool and reused by a different logical request before the setting is cleared, leaking tenant context across requests. This is why **every** tenant-scoped database call in this project must go through the `withTenantContext` wrapper (§5.2), which explicitly wraps `BEGIN ... SET LOCAL (via set_config) ... COMMIT` around a single borrowed connection. This is a hard rule, enforced via a custom ESLint rule and code-review checklist starting in Phase 1, Task 1.4.

### 4.4 Entity relationship summary

```mermaid
erDiagram
    TENANTS ||--o{ USERS : employs
    TENANTS ||--|| TENANT_SETTINGS : configures
    TENANTS ||--o{ DEPARTMENTS : contains
    TENANTS ||--o{ DOCTOR_AFFILIATIONS : hosts
    TENANTS ||--o{ APPOINTMENTS : manages
    TENANTS ||--o{ LAB_REQUESTS : processes
    TENANTS ||--o{ PHARMACY_ORDERS : fulfills

    USERS ||--o| PATIENT_PASSPORTS : owns
    PATIENT_PASSPORTS ||--o{ PASSPORT_CONSENT_GRANTS : authorizes
    USERS ||--o{ DOCTOR_AFFILIATIONS : practices_at
    DEPARTMENTS ||--o{ DOCTOR_AFFILIATIONS : assigns

    USERS ||--o{ APPOINTMENTS : books
    USERS ||--o{ APPOINTMENTS : conducts

    APPOINTMENTS ||--o| PRESCRIPTIONS : yields
    PRESCRIPTIONS ||--o{ PRESCRIPTION_MEDICINES : contains
    PRESCRIPTIONS ||--o{ PHARMACY_ORDERS : triggers
    PHARMACY_ORDERS ||--o{ DOSE_LOGS : schedules

    USERS ||--o{ LAB_REQUESTS : requests

    APPOINTMENTS ||--o{ BILLING_RECEIPTS : generates
    LAB_REQUESTS ||--o{ BILLING_RECEIPTS : generates
    PHARMACY_ORDERS ||--o{ BILLING_RECEIPTS : generates
    BILLING_RECEIPTS ||--o{ RECEIPT_ITEMS : itemizes

    USERS ||--o{ EMERGENCY_DISPATCHES : triggers
    EMERGENCY_DISPATCHES }o--|| AMBULANCE_UNITS : dispatches
```

### 4.5 Database migration workflow **[NEW]**

Since the source documents specify final-state DDL but not how a team gets there safely:

1. Use a numbered, timestamped migration tool (`node-pg-migrate` or Knex migrations) — never hand-run ad hoc SQL against shared environments.
2. One migration file per numbered table/feature above (`0001_tenants.sql` ... `0014_notifications.sql`, `0015_rls_policies.sql`), matching the DDL blocks above 1:1, so `git blame` on a migration file maps directly to a section of this document.
3. Every migration must have a paired `down` migration, even though this project does not expect to roll back production migrations often — CI runs `up` then `down` then `up` again on every PR to catch irreversible mistakes early.
4. Seed data (§Phase 1 Task 1.5) is a **separate** script from schema migrations, runnable idempotently against a fresh database for local dev and staging, never against production.

---

## 5. Backend Deep-Dive Implementation

### 5.1 Layered architecture

```
+-------------------------------------------------------------------+
|  HTTP / WebSocket Layer — Express routes, Socket.io handlers,     |
|  middleware (auth, tenant, idempotency, rate limit)               |
+-------------------------------------------------------------------+
                                  |
+-------------------------------------------------------------------+
|  Controller Layer — request validation (Zod), response formatting |
+-------------------------------------------------------------------+
                                  |
+-------------------------------------------------------------------+
|  Service Layer — business logic, AI engine calls, queue/lock math |
+-------------------------------------------------------------------+
                                  |
+-------------------------------------------------------------------+
|  Repository Layer — parameterized SQL, transactions               |
+-------------------------------------------------------------------+
                                  |
+-------------------------------------------------------------------+
|  Database & Cache — PostgreSQL pool (via PgBouncer), Redis        |
+-------------------------------------------------------------------+
```

Every module (`auth`, `tenant`, `passport`, `queue`, `emergency`, `triage`, `lab`, `pharmacy`, `billing`, `notifications`) follows this same four-layer shape. Controllers never touch SQL directly; services never touch `req`/`res` directly. This separation is what makes the ESLint rule in §4.3 enforceable — a lint rule can statically check "no raw `pool.query` outside `*.repository.js`."

### 5.2 Tenant Context Middleware & PgBouncer-Safe Wrapper

```javascript
// backend/src/middleware/tenant.middleware.js
export const enforceTenantContext = async (req, res, next) => {
  try {
    let tenantId = null;
    if (req.user?.role === 'super_admin') {
      // Only super_admin may switch tenant context, and only via a validated header
      const headerTenant = req.headers['x-tenant-id'];
      if (headerTenant) {
        const tenant = await tenantRepository.findActiveById(headerTenant);
        if (!tenant) return res.status(404).json({ error: 'Tenant invalid or deactivated' });
        tenantId = tenant.id;
      }
    } else if (req.user) {
      // Every other role: tenant context comes ONLY from the JWT claim, never a client header
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
    await client.query('SELECT set_config($1, $2, true)', ['app.current_tenant_id', tenantId]); // parameterized, never interpolated
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

### 5.3 Idempotency Middleware

```javascript
// backend/src/middleware/idempotency.middleware.js
export const idempotencyGuard = (resourceCheckFn) => async (req, res, next) => {
  const key = req.headers['idempotency-key'];
  if (!key) return res.status(400).json({ error: 'Idempotency-Key header required' });

  const existing = await resourceCheckFn(key);
  if (existing) {
    return res.status(200).json({ idempotent: true, data: existing }); // return original, never duplicate
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
The booking service persists `req.idempotencyKey` into `appointments.idempotency_key` in the **same transaction** that allocates the token and creates the pending billing receipt, so a retried request with the same key never double-books or double-charges.

### 5.4 Atomic Queue & Lite Token Engine

```javascript
// backend/src/modules/queue/queue.service.js
export class QueueService {
  static async allocateRegularToken(tenantId, doctorId, date) {
    const redisKey = `token_counter:${tenantId}:${doctorId}:${date}`;
    const tokenNumber = await redis.incr(redisKey);
    await redis.expire(redisKey, 172800); // 48h TTL, past the appointment date
    try {
      await withTenantContext(tenantId, (client) =>
        appointmentRepository.insertAppointment(client, { tenantId, doctorId, date, tokenNumber })
      );
      return tokenNumber;
    } catch (err) {
      if (isUniqueViolation(err)) {
        // Extremely rare: Redis counter desynced from DB (e.g. after a Redis flush)
        return allocateTokenFallback(tenantId, doctorId, date);
      }
      throw err;
    }
  }

  static async allocateTokenFallback(tenantId, doctorId, date) {
    return withTenantContext(tenantId, async (client) => {
      const { rows } = await client.query(
        `SELECT COALESCE(MAX(token_number), 0) + 1 AS next_token
         FROM appointments
         WHERE tenant_id = $1 AND doctor_id = $2 AND appointment_date = $3
           AND type = 'REGULAR'
         FOR UPDATE`,   // Postgres syntax only — never SQL Server's WITH (UPDLOCK, HOLDLOCK)
        [tenantId, doctorId, date]
      );
      return rows[0].next_token;
    });
  }

  // Lite Appointment: only offered when patient has a COMPLETED appointment with
  // this doctor in the last 90 days — a follow-up mechanism, not a queue-skip for new visits.
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
        // If already taken (rare — two lite bookings before the next regular token advances),
        // step forward in 0.1 increments until a free slot is found.
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

### 5.5 PostGIS Emergency Dispatch & Atomic Driver Lock

```javascript
// backend/src/modules/emergency/emergency.service.js
export class EmergencyService {
  static async triggerEmergency(patientId, location) {
    // PostGIS 10km spatial search
    const candidates = await ambulanceRepository.findNearby(location, 10000);
    const dispatch = await emergencyRepository.create({ patientId, location, candidates });

    if (candidates.length === 0) {
      // Zero-candidate fallback: escalate immediately, do NOT wait for the 15s driver timeout
      // (that timeout assumes at least one candidate was broadcast to)
      await emergencyRepository.escalateToExternalDispatch(dispatch.id); // 108/102
      return dispatch;
    }

    socketService.broadcastToDrivers(candidates, dispatch);
    scheduleTimeoutEscalation(dispatch.id, 15_000); // 15s no-acceptance -> escalate
    return dispatch;
  }

  static async acceptDispatch(dispatchId, driverId, tenantId) {
    // Single atomic conditional UPDATE — first driver to hit this wins, no race condition
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
    socketService.notifyLosingCandidates(dispatchId, driverId); // tell other drivers to stand down
    return { accepted: true, dispatch: rows[0] };
  }
}
```

Underlying spatial query:
```sql
SELECT id, driver_id, ST_Distance(current_location, ST_MakePoint($1, $2)::geography) AS distance_meters
FROM ambulance_units
WHERE is_available = TRUE AND ST_DWithin(current_location, ST_MakePoint($1, $2)::geography, 10000)
ORDER BY distance_meters ASC;
```

### 5.6 Hybrid Clinical Decision Support (CDS) Engine

Two AI-adjacent systems exist in this product and **must stay architecturally separate** — conflating them was the single most dangerous mistake in early drafts:

| System | Role | Engine | Failure mode if wrong |
|---|---|---|---|
| **CDS Warning Engine** (dosage, contraindication, pediatric/allergy checks) | **Primary, blocking** | Deterministic rule engine | Patient harm — must never be probabilistic |
| **AI Symptom Triage** (department routing) | Advisory, non-blocking | LLM (Gemini/GPT-4o-mini), structured JSON | Wrong department suggestion — recoverable, low harm |
| **AI Queue Wait-Time Predictor** | Advisory | Regression model | Wrong ETA — inconvenience, not harm |

```javascript
// backend/src/modules/triage/cds.service.js
export class HybridCDSEngine {
  static async evaluatePrescription(patientPassport, newMedicines) {
    const hardWarnings = [];

    // 1. DETERMINISTIC SAFETY RULES — primary, blocking gate
    for (const med of newMedicines) {
      if (patientPassport.age < 12 && ['aspirin', 'tetracycline'].some(d => med.name.toLowerCase().includes(d))) {
        hardWarnings.push(`CRITICAL CDS ALERT: ${med.name} is contraindicated in pediatric patients (<12 years).`);
      }
      if (patientPassport.allergies.some(a => med.name.toLowerCase().includes(a.toLowerCase()))) {
        hardWarnings.push(`CRITICAL ALLERGY ALERT: Patient is allergic to ${med.name}.`);
      }
      // Additional deterministic rules to implement in Phase 5: dosage-threshold checks,
      // radiation/fasting prerequisites for lab-linked prescriptions, drug-drug interaction matrix.
    }

    if (hardWarnings.length > 0) {
      return { status: 'REJECTED', warnings: hardWarnings, source: 'DETERMINISTIC_ENGINE' };
    }

    // 2. LLM advisory explanation layer — never originates or suppresses a warning
    const aiSummary = await AIService.generateMedicationSummary(newMedicines);
    return { status: 'APPROVED', warnings: [], aiSummary, source: 'HYBRID_ENGINE' };
  }
}
```
The deterministic engine remains system-of-record for prescription warnings; an LLM may generate a plain-language *explanation* of a triggered rule, never originate or suppress one. The UI (§Phase 6 Doctor Desk) reinforces this: a `REJECTED` result is a blocking red banner the doctor cannot dismiss without editing the prescription; an `APPROVED` result shows the AI summary as a dismissible note explicitly labeled "AI explanation, not a safety check."

### 5.7 MFA Secret Encryption (AES-256-GCM)

```javascript
// backend/src/utils/mfaCrypto.js
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

### 5.8 Socket.io Handshake Auth & Room Authorization

```javascript
// backend/src/services/socket.authorization.js
export async function authorizeRoomAccess(user, room) {
  const [type, ...rest] = room.split(':');

  if (type === 'queue') {
    const [tenantId, doctorId] = rest;
    if (user.role === 'patient') {
      return appointmentRepository.hasActiveAppointment(user.id, doctorId, tenantId);
    }
    return user.tenantId === tenantId; // staff: only their own tenant's queue room
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
  return false; // deny by default for any unrecognized room type
}

// On connection:
io.use(async (socket, next) => {
  try {
    const decoded = verifyJwt(socket.handshake.auth.token);
    socket.user = decoded;
    next();
  } catch { next(new Error('Unauthorized')); }
});

socket.on('join', async ({ room }) => {
  const isAuthorized = await authorizeRoomAccess(socket.user, room);
  if (!isAuthorized) return socket.emit('error', 'Unauthorized room access');
  socket.join(room);
});
```
Room structure: `tenant:{hospitalId}` (hospital-wide alerts), `queue:{tenantId}:{doctorId}` (live token advancement), `emergency:{dispatchId}` (live GPS + status updates).

### 5.9 AI Crowd Status Engine

```javascript
// backend/src/modules/hospitals/crowdStatus.service.js
export const computeCrowdStatus = (activeTokensToday, dailyCapacity) => {
  const ratio = activeTokensToday / dailyCapacity;
  if (ratio < 0.3) return { label: 'Low', code: 'LOW', color: 'GREEN' };
  if (ratio < 0.7) return { label: 'Moderate', code: 'MODERATE', color: 'YELLOW' };
  return { label: 'High', code: 'HIGH', color: 'RED' };
};

// Cached in Redis, 60s TTL — this backs a browsing/discovery list, not a live ticket, so
// per-second freshness is unnecessary and would waste DB load.
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
Uses `tenant_settings.daily_token_capacity_per_doctor` (aggregated across active doctors) rather than a hardcoded constant.

### 5.10 Notification Fan-Out Architecture (BullMQ)

```
[ Application Event ] (e.g., Lab Report COMPLETED)
         |
         v
[ notifications table INSERT ]  <-- always happens first, in-app bell/inbox is authoritative
         |
         v
[ Notification Publisher ] ---> Pushes job to BullMQ Queue
         |
         v
[ BullMQ Worker Process ]
         |
         +---> WebSocket  ---> emits to the user's connected socket, if online
         +---> Push Engine ---> FCM / APNS
         +---> SMS Gateway ---> Twilio / MSG91  (checks notification_consents first — TRAI)
         +---> Email Queue ---> SendGrid HTML email (checks notification_consents first)
```
Critical events (appointment, emergency) always fire in-app and push regardless of SMS/email consent status; only SMS/email channels are consent-gated per TRAI regulations.

### 5.11 Payment Gateway Integration **[NEW]**

No source document specifies an actual payment integration despite constant references to "Patient pays." This blueprint specifies:

- **Provider:** Razorpay (India-first, UPI/card/netbanking support, webhook-based confirmation) — selected over Stripe for native UPI support, which is the dominant payment method for this user base.
- **Flow:** `POST /api/v1/payments/create-order` (server creates a Razorpay order tied to the pending `appointments`/`lab_requests`/`pharmacy_orders` row) → client completes payment via Razorpay Checkout → Razorpay webhook (`POST /api/v1/payments/webhook`, signature-verified) flips the resource to its paid state (`CONFIRMED`, `PAID`, etc.) in the same transaction that inserts the `billing_receipts` row.
- **Idempotency:** the webhook handler is itself idempotent, keyed on Razorpay's `payment_id`, since webhook delivery is at-least-once by design.
- **Failure handling:** if payment fails or times out, the underlying resource remains in `PENDING_PAYMENT`; a BullMQ delayed job expires unpaid appointments after 15 minutes (analogous to the existing 48h pharmacy-order auto-expiry pattern in §Phase 5) to free the token slot for other patients.
- **Refunds:** admin-initiated only in v1 (no self-service patient refund flow), logged to `audit_logs`.

---

## 6. API Surface (Complete Matrix)

| Method | Endpoint | Auth | Request | Response | SLA (p95) | Notes |
|---|---|---|---|---|---|---|
| `POST` | `/api/v1/auth/register` | Public | `{email,password,full_name,role}` | `{user, token}` | <100ms | Runs pre-tenant-context; uses `BYPASSRLS` lookup path |
| `POST` | `/api/v1/auth/login` | Public | `{email,password}` | `{user, accessToken, refreshToken}` | <100ms | Issues access + refresh token pair |
| `POST` | `/api/v1/auth/refresh` | Refresh token | `{refreshToken}` | `{accessToken, refreshToken}` | <100ms | Rotates token; reuse triggers family revocation |
| `POST` | `/api/v1/auth/mfa/setup` | Session | `{}` | `{qrCodeUrl, secret}` | <100ms | Generates TOTP secret, encrypted before storage |
| `POST` | `/api/v1/auth/mfa/verify` | Session | `{code}` | `{user, accessToken}` | <100ms | Decrypts `mfa_secret_encrypted` in-memory only |
| `GET` | `/api/v1/hospitals/nearby` | Auth | `?lat&lng&radius` | `[{id,name,distance,crowdStatus}]` | <100ms | PostGIS distance query + 60s-cached crowd status |
| `POST` | `/api/v1/triage/analyze` | Auth | `{symptoms}` | SSE stream `{department}` | <3.0s | Department-only output, safety disclaimer always rendered |
| `POST` | `/api/v1/appointments/book` | Patient | Header `Idempotency-Key` | `{appointmentId, tokenNumber}` | <100ms | Regular token via `allocateRegularToken` |
| `POST` | `/api/v1/appointments/book-lite` | Patient | Header `Idempotency-Key` | `{appointmentId, tokenNumber: 15.5}` | <100ms | Only shown if `COMPLETED` appt with doctor in last 90 days |
| `GET` | `/api/v1/queue/live/:doctorId` | Auth | `?date` | `{currentToken, yourToken, patientsAhead, estimatedWaitMinutes}` | <100ms | REST snapshot; live updates via `queue:{tenantId}:{doctorId}` socket room |
| `POST` | `/api/v1/emergency/trigger` | Patient | `{lat,lng}` | `{dispatchId, status}` | <100ms | Zero-candidate path auto-escalates to 108/102 |
| `POST` | `/api/v1/emergency/accept/:id` | Driver | `{}` | `{accepted, dispatch}` | <50ms | Atomic conditional update |
| `GET` | `/api/v1/passport/timeline` | Auth + Consent | `{}` | `{passportId, timeline:[...]}` | <100ms | Requires active `passport_consent_grants` row |
| `POST` | `/api/v1/passport/consent/grant` | Patient | `{doctorId, grantType, expiresAt}` | `{grantId}` | <100ms | |
| `POST` | `/api/v1/passport/consent/revoke/:id` | Patient | `{}` | `{revoked:true}` | <100ms | Takes effect on next request, not retroactively mid-consult |
| `POST` | `/api/v1/prescriptions/sign` | Doctor | `{appointmentId, medicines}` | `{prescriptionId, cdsEvaluation}` | <100ms | Blocked by `REJECTED` CDS result |
| `POST` | `/api/v1/lab/reports/upload` | Lab Tech | FormData `file` | `{reportUrl, status}` | <5.0s | ClamAV-scanned, magic-byte-checked multipart upload |
| `POST` | `/api/v1/pharmacy/orders/:id/confirm` | Patient | `{}` | `{orderId, status:"RECEIVED"}` | <100ms | Starts 48h auto-expiry timer |
| `PATCH` | `/api/v1/dose-logs/:id/taken` | Patient | `{}` | `{doseLogId, takenAt}` | <100ms | Marks a reminder as taken |
| `POST` | `/api/v1/payments/create-order` | Auth | `{resourceType, resourceId}` | `{razorpayOrderId}` | <100ms | See §5.11 |
| `POST` | `/api/v1/payments/webhook` | Signature-verified | Razorpay payload | `200 OK` | <100ms | Idempotent on `payment_id` |
| `GET` | `/api/v1/notifications` | Auth | `?unreadOnly` | `[{id,title,body,isRead}]` | <100ms | |
| `PATCH` | `/api/v1/notifications/:id/read` | Auth | `{}` | `{id, isRead:true}` | <100ms | |
| `GET` | `/api/v1/admin/audit-logs` | Hospital Admin | `?filters` | `[{action,actor,timestamp}]` | <100ms | Tenant-scoped via RLS, append-only source |
| `POST` | `/api/v1/admin/queue/override` | Hospital Admin | `{appointmentId, newToken}` | `{updated:true}` | <100ms | Writes `audit_logs` row per override |
| `POST` | `/api/v1/super-admin/tenants` | Super Admin | `{name, code, address,...}` | `{tenantId}` | <100ms | Hospital onboarding |

### 6.1 API integration order for frontend work **[NEW]**

Since no source document sequences frontend-to-backend dependency explicitly, the build order is:

1. Auth endpoints (register/login/refresh/MFA) — nothing else can be tested without a session.
2. Hospital discovery + tenant/department read endpoints — needed before booking screens can render real data.
3. Passport read + consent endpoints — needed before any doctor-facing screen can show patient context.
4. Appointment booking + live queue endpoints — the core patient loop.
5. Emergency trigger/accept endpoints + Socket.io rooms — independent of booking, can be built in parallel by a second engineer/pair once auth exists.
6. Prescription/CDS, lab, pharmacy endpoints — depend on appointments existing.
7. Billing + payments — depend on appointments/lab/pharmacy existing (billing is a *consequence* of those, never built first).
8. Notifications + admin/audit endpoints — can be built anytime after auth, but are lowest priority for the patient-facing MVP.

This ordering directly drives the Phase numbering in §13.

---

## 7. Frontend Architecture & Design System

### 7.1 Design language decision (resolves §0.2 conflict #1)

**Light theme only. Zero dark mode in v1.** Organic soft radii, elevated floating cards, zero boxy patterns.

```css
/* src/styles/tokens.css */
:root {
  /* Brand Core Palette */
  --brand-primary: #03A6A1;       /* Deep Teal — trustworthy clinical primary */
  --brand-secondary: #FFE3BB;     /* Soft warm cream / sand accent */
  --brand-accent: #FFA673;        /* Warm coral accent */
  --brand-cta: #FF4F0F;           /* Vibrant crimson-orange — SOS & primary CTAs */

  /* Light Theme Canvas & Surfaces (zero dark mode) */
  --bg-canvas: #F8FAFC;
  --bg-surface: #FFFFFF;
  --bg-subsurface: #FFF8F0;
  --bg-card-hover: #F1F5F9;

  /* Organic soft border & shadow tokens (zero sharp boxy patterns) */
  --border-subtle: #E2E8F0;
  --border-active: #03A6A1;
  --shadow-sm: 0 2px 8px rgba(3, 166, 161, 0.04);
  --shadow-md: 0 10px 30px -10px rgba(3, 166, 161, 0.08);
  --shadow-lg: 0 20px 40px -15px rgba(3, 166, 161, 0.12);
  --shadow-sos: 0 12px 36px rgba(255, 79, 15, 0.25);

  /* Organic radii (no sharp 0px corners) */
  --r-sm: 12px;
  --r-md: 20px;
  --r-lg: 28px;
  --r-pill: 9999px;

  /* Typography */
  --text-main: #0F172A;
  --text-muted: #475569;
  --text-subtle: #94A3B8;

  /* Functional status badges */
  --status-low: #10B981;    /* Low crowd */
  --status-mod: #F59E0B;    /* Moderate crowd */
  --status-high: #EF4444;   /* High crowd */

  --font-sans: 'Outfit', -apple-system, BlinkMacSystemFont, sans-serif;
  --font-mono: 'Fira Code', monospace;
  --touch-target-min: 48px;
}
```

**Anti-boxy principles (mandatory, checked in every PR touching UI):**
1. Zero sharp corners — cards/inputs/buttons/modals use `--r-md` to `--r-lg`.
2. Elevated floating surfaces — no thick borders; multi-layered soft shadows (`--shadow-md`) instead.
3. Pill-shaped interactive elements — buttons, status pills, department tags use `--r-pill`.
4. Breathing space — 24–32px container padding, generous gaps.

### 7.2 GSAP animation engine

```javascript
// src/utils/gsapAnimations.js
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

export const initScrollReveals = (selector = '.gsap-reveal') => {
  document.querySelectorAll(selector).forEach((el) => {
    gsap.fromTo(el, { opacity: 0, y: 20 }, {
      opacity: 1, y: 0, duration: 0.5, ease: 'power2.out',
      scrollTrigger: { trigger: el, start: 'top 88%', toggleActions: 'play none none reverse' },
    });
  });
};

export const animateCtaPulse = (target) => gsap.to(target, {
  scale: 1.025, duration: 0.9, repeat: -1, yoyo: true, ease: 'sine.inOut',
});
```
Animations are lightweight and non-distracting by rule: no animation may exceed ~0.9s duration, no more than one concurrent pulse animation per screen (the SOS button), and `initScrollReveals` must be idempotent (safe to call again on route change without double-binding).

### 7.3 State management strategy **[NEW — explicit ownership split]**

No source document draws this line explicitly beyond a diagram; this is the concrete rule every engineer follows:

| State category | Owner | Examples | Why |
|---|---|---|---|
| **Client-only, ephemeral** | Zustand | Auth session (JWT in memory, not localStorage — see §10), active tenant context, mobile nav drawer open/closed, SOS local UI state (`IDLE`→`CONFIRM_MODAL`), theme constants | Doesn't need caching/refetch semantics; Zustand avoids boilerplate for state no server owns |
| **Server-derived, cacheable** | TanStack Query v5 | Hospital directory, doctor schedules, passport timeline, billing history, notification list | Needs automatic caching, background refetch, invalidation-on-mutation (e.g. booking invalidates the queue-position query) |
| **Real-time, push-driven** | Socket.io client + a thin Zustand slice it writes into | Live queue ticker, ambulance GPS stream, notification bell counter | Data arrives unsolicited from the server; React Query's pull model doesn't fit, but the *last known value* still needs to be read by components, hence writing into a small Zustand slice on each socket event |

**Hard rule:** a socket event handler never calls `setState` on more than one Zustand slice, and React Query cache is never mutated directly from a socket handler — if a socket event should invalidate a cached query (e.g. `queue:updated` should invalidate `/queue/live/:doctorId` REST snapshot), call `queryClient.invalidateQueries(...)`, don't hand-patch the cache.

### 7.4 Device-tier responsive strategy

```
+-----------------------------------------------------------------------------------+
|  1. SMARTPHONES (< 640px) [Patients & public users]                               |
|  - Fixed thumb-zone bottom nav (Home, Passport, Queue, Emergency SOS)             |
|  - Minimum 48x48px tap targets with tactile press-scale feedback                  |
|  - Slide-up bottom sheets for triage input, filters, SOS confirmation             |
|  - Swipe gestures to dismiss sheets/notifications                                 |
+-----------------------------------------------------------------------------------+
|  2. TABLETS (640-1024px) [Doctors & clinical staff]                               |
|  - Split-screen: queue list (35%) + EHR/passport (65%)                            |
|  - Touch-optimized prescription builder & lab requester buttons                   |
|  - Dual orientation support (ward rounds)                                         |
+-----------------------------------------------------------------------------------+
|  3. DESKTOPS (> 1024px) [Hospital admins & back-office]                           |
|  - Multi-column dashboards, sticky-header data tables, bulk selection tools       |
+-----------------------------------------------------------------------------------+
```

### 7.5 Page-by-page UI map (14 screens)

```
                                  healthcare+ Light UI Application
                                                 |
    +--------------------------------------------+--------------------------------------------+
    v                                                                                          v
[ PATIENT PORTAL & PUBLIC APP ]                                            [ STAFF & CLINICAL CONSOLES ]
 1. Landing Page & Public Directory                                         9. Doctor Desk Console (split tablet view)
 2. Auth & Role Onboarding Modal                                           10. Ambulance Driver App (Uber/Rapido style)
 3. User Home Dashboard (AI triage & 4-nearest hospitals)                  11. Pharmacy Fulfillment Console
 4. Personal Patient Dashboard (queue ticket & dose checkboxes)            12. Laboratory Diagnostics Console
 5. Universal Healthcare Passport (timeline & consent grants)              13. Hospital Admin Panel
 6. Hospital Workspace (doctors, pharmacy, labs, billing)                  14. Platform Super Admin Console
 7. Doctor Booking Wizard (regular & Lite #15.5 appointments)
 8. Uber/Rapido-style live Emergency SOS tracking map
```

| # | Screen | Key elements |
|---|---|---|
| 1 | Landing & Public Directory | Pristine canvas, floating cards, AI triage teaser, hospital search |
| 2 | Auth & Role Onboarding | Role selector pills, TOTP MFA 6-digit auto-focus input |
| 3 | User Home Dashboard | Warm-cream AI Assistant banner, 3s-hold SOS button, 4-nearest-hospital grid with crowd badges |
| 4 | Personal Patient Dashboard | View switcher, live queue ticket, dose_logs checklist, lab PDFs, billing history |
| 5 | Universal Healthcare Passport | Cross-hospital timeline, ABHA badge, consent grant toggles |
| 6 | Hospital Workspace | Branding header, pill tabs: Doctors / Appointments / Pharmacy / Laboratory / Billing / Notifications |
| 7 | Doctor Booking Wizard | Department→Doctor→Slot picker, Lite Appointment toggle |
| 8 | Emergency SOS Live Map | Searching → Driver Accepted → Live GPS/ETA → ER Pre-Notified |
| 9 | Doctor Desk | Queue list (35%) + Passport/e-Prescription (65%) with CDS banners |
| 10 | Ambulance Driver Console | Online/Available toggle, 15s dispatch sheet, turn-by-turn nav |
| 11 | Pharmacy Fulfillment | Status stepper: Received → Packed → Paid → Completed |
| 12 | Laboratory Console | Test list, Sample Collected toggle, ClamAV-scanned PDF drop zone |
| 13 | Hospital Admin Panel | Staff, fees, queue overrides, lab/pharmacy pricing, ER monitor, revenue analytics |
| 14 | Platform Super Admin | Hospital onboarding, network capacity analytics |

### 7.6 Accessibility requirements **[NEW]**

Referenced in none of the seven source documents despite being a hard requirement for a healthcare product used by an older, more varied population than a typical consumer app:

- **Baseline:** WCAG 2.1 AA across all patient-facing screens (1–8); staff consoles (9–14) target AA where practical but may relax color-contrast-on-dense-tables requirements with documented exceptions.
- **Touch targets:** 48×48px minimum (already a design-token rule, §7.1) doubles as a motor-accessibility requirement, not just a mobile-ergonomics one.
- **Color is never the only signal:** crowd status badges (🟢🟡🔴) always pair color with text label ("Low"/"Moderate"/"High") and, on the SOS flow specifically, with an icon — color-blind users must be able to read hospital load and dispatch state without color.
- **Screen reader support:** all icon-only buttons (bottom nav, SOS hold button, queue ticker) require `aria-label`; the SOS 3-second hold interaction requires an accessible alternative (double-tap-and-confirm) for users who cannot sustain a physical hold gesture — this is a Phase 6 task, not deferred to polish.
- **Forms:** every input has a visible, associated `<label>` (no placeholder-as-label anti-pattern); error messages are programmatically associated via `aria-describedby`.
- **Motion:** GSAP animations respect `prefers-reduced-motion` — `initScrollReveals` checks this media query and short-circuits to a static, fully-visible state when set.
- **Testing:** automated axe-core checks in CI on every PR touching `src/components` or `src/pages`; manual screen-reader pass (NVDA + VoiceOver) required before each phase's UI work is marked complete.

### 7.7 Error-handling contract **[NEW]**

- **HTTP error shape (all endpoints):** `{ error: string, code: string, details?: object }` with standard HTTP status codes; the frontend never parses `error` strings for logic — it switches on `code`.
- **Frontend error boundaries:** one top-level React error boundary per major route group (Patient Portal, Staff Consoles, Admin) so a crash in one console doesn't blank the whole app; each boundary renders a light-theme-consistent fallback with a "Reload" action and auto-reports to Sentry (§16).
- **Network/offline handling:** React Query's default retry (3x, exponential backoff) is used for GET requests; mutations (booking, prescription signing) do **not** auto-retry silently — a failed mutation surfaces an explicit retry button, since silent-retry-on-a-write is how double-bookings happen even with idempotency keys as a backstop.
- **SOS flow specifically:** because a failed emergency request is not an acceptable silent failure, the SOS trigger call has its own dedicated handling — on network failure, the UI immediately surfaces a "Call 108 directly" fallback button (tel: link) rather than a generic error toast, and retries the trigger call in the background without blocking that fallback's visibility.

### 7.8 Frontend folder structure

```
frontend/
├── src/
│   ├── assets/                  # Fonts, icons, static images
│   ├── components/
│   │   ├── ui/                  # Buttons, Cards, Inputs, Modals, Badges (primitives)
│   │   ├── layout/               # Sidebar, Header, BottomNav, PageContainer
│   │   ├── passport/             # Medical Timeline, Allergy Badges, Consent toggles
│   │   ├── queue/                 # Live Token Ticket, Delay Predictor
│   │   └── emergency/             # Uber-style Map Tracker
│   ├── context/                  # Socket Context Provider
│   ├── hooks/                    # useQueueSocket, useGeoLocation, useAuth, usePrefersReducedMotion
│   ├── pages/
│   │   ├── Landing/
│   │   ├── PatientDashboard/
│   │   ├── HospitalExplorer/
│   │   ├── DoctorConsole/
│   │   ├── LabConsole/
│   │   ├── PharmacyConsole/
│   │   ├── EmergencySOS/
│   │   └── AdminDashboard/
│   ├── services/                 # Axios API instance & endpoint wrappers
│   ├── store/                    # Zustand stores (useAuthStore, useQueueStore, useEmergencyStore)
│   ├── styles/                   # tokens.css & global CSS
│   ├── utils/                    # gsapAnimations.js
│   ├── App.jsx                   # Router setup
│   └── main.jsx
├── index.html
├── vite.config.js
└── package.json
```

### 7.9 Backend folder structure

```
backend/
├── src/
│   ├── config/
│   │   ├── database.js           # PostgreSQL pool config
│   │   ├── redis.js              # Redis client & PubSub
│   │   └── environment.js        # Env var validation (Zod) — see §11 env schema
│   ├── constants/                 # Roles, status enums, error codes
│   ├── middleware/
│   │   ├── auth.middleware.js     # JWT verification & RBAC check
│   │   ├── tenant.middleware.js   # enforceTenantContext (§5.2)
│   │   ├── idempotency.middleware.js
│   │   ├── error.middleware.js    # Centralized error handling (§7.7 shape)
│   │   └── rateLimiter.js
│   ├── modules/
│   │   ├── auth/                  # routes, controller, service
│   │   ├── tenant/
│   │   ├── passport/
│   │   ├── queue/
│   │   ├── emergency/
│   │   ├── triage/                # AI triage + Hybrid CDS engine
│   │   ├── lab/
│   │   ├── pharmacy/
│   │   ├── billing/
│   │   ├── payments/
│   │   └── notifications/
│   ├── repository/
│   │   └── withTenantContext.js
│   ├── services/
│   │   ├── socket.service.js
│   │   ├── socket.authorization.js
│   │   ├── ai.service.js          # Gemini/OpenAI client
│   │   ├── notification.service.js # BullMQ workers
│   │   └── storage.service.js      # S3/local PDF storage
│   ├── utils/                      # Pino logger, mfaCrypto.js
│   └── app.js
├── migrations/                     # numbered migration files, §4.5
├── seed/                           # idempotent demo-data seed script, §Phase 1
├── Dockerfile
└── package.json
```

---

## 8. Role-Based Workflows & State Machines

### 8.1 Emergency SOS state machine

```
IDLE
  → (hold 3s, or double-tap-confirm for accessibility) → CONFIRM_MODAL
      → (confirm) → LOCATING                    [HTML5 Geolocation]
          → (coords acquired) → BROADCASTING     [PostGIS 10km search]
              → (0 candidates) → ESCALATED_108    [terminal, shown to patient]
              → (>=1 candidate) → candidates notified via socket
                  → (driver accepts, atomic lock wins) → ACCEPTED
                      → EN_ROUTE_PATIENT → PATIENT_PICKED → ARRIVED_HOSPITAL  [terminal]
                  → (15s no acceptance) → ESCALATED_108   [terminal]
      → (cancel before driver accepts) → CANCELLED  [terminal, requires reason]
```
Patient-facing copy: *"Searching for nearby ambulances…"* → *"Driver Accepted! Ramesh Kumar (KA-01-EA-1234)"* → live map with dynamic ETA → *"Destination Hospital ER Pre-Notified."* Cancellation after driver assignment (`EN_ROUTE_PATIENT` or later) is **not** exposed in the UI — only pre-acceptance cancellation is allowed, to avoid a dispatched driver being silently abandoned mid-route.

### 8.2 Pharmacy order state machine

```
Doctor signs prescription
   → Patient confirms purchase [pharmacy_orders: PENDING_CONFIRMATION]
       → Pharmacy staff marks RECEIVED
           → Pharmacy staff marks PACKED  → notification sent to patient
               → Patient pays (if not pre-paid) → PAID
                   → Patient collects → COMPLETED
                       → reminders_activated = TRUE
                       → dose_logs schedule generated from prescription_medicines
```
Unconfirmed orders auto-expire to `CANCELLED` after 48h via a BullMQ delayed job, preventing an indefinite `PENDING_CONFIRMATION` backlog.

### 8.3 Laboratory request state machine

```
Doctor creates request [lab_requests: REQUESTED]
   → Patient views cost, confirms & pays → PAID
       → Lab marks SAMPLE_COLLECTED
           → Lab marks PROCESSING
               → Lab uploads PDF (ClamAV scan + magic-byte check) → COMPLETED
                   → report_file_url attached
                   → in-app notification fired to doctor and patient
                   → visible immediately in Healthcare Passport timeline
```

### 8.4 Appointment lifecycle

```
PENDING_PAYMENT → (payment webhook confirms) → CONFIRMED → (doctor calls token) → IN_PROGRESS
   → (consultation ends) → COMPLETED
   → (patient doesn't show) → NO_SHOW   [distinct from CANCELLED — enables no-show-rate analytics]
   → (before CONFIRMED, or admin override) → CANCELLED
```

### 8.5 RBAC matrix

| Role | Hospital Settings | Book Appointment | Manage Queue | View Passport | Prescribe | Lab Upload | Dispense Meds | Emergency Dispatch | System Analytics |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Super Admin** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ (network-wide) |
| **Hospital Admin** | ✅ | ❌ | ✅ (override, audited) | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ (own tenant) |
| **Doctor** | ❌ | ❌ | ✅ | ✅ (granted, consent-scoped) | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Receptionist** | ❌ | ✅ (on behalf of walk-ins) | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Patient** | ❌ | ✅ | ✅ (own) | ✅ (own) | ❌ | ❌ | ❌ | ✅ (trigger) | ❌ |
| **Lab Tech** | ❌ | ❌ | ❌ | ❌ (unless granted) | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Pharmacist** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| **Nurse** | ❌ | ❌ | ✅ (view/assist) | ✅ (granted) | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Ambulance Driver** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ (accept) | ❌ |

### 8.6 Authentication & authorization flow

```
[ Client ] ---> POST /auth/login { email, password }
   |
   v
[ Auth Controller ] ---> Verify Argon2id hash
   |
   v
[ Token Engine ] ---> Issues:
                        1. Access Token  (RS256 JWT, 15 min)
                        2. Refresh Token (HttpOnly, SameSite cookie, 7 days, token_family_id)
   |
   v
[ If MFA enabled ] ---> Session token issued instead; POST /auth/mfa/verify required before access token
   |
   v
[ Subsequent requests ] ---> Header: Authorization: Bearer <accessToken>
   |
   v
[ Auth Middleware ] ---> Decodes JWT, validates active session in Redis, sets req.user + req.tenantId
   |
   v
[ Refresh flow ] ---> Single-use rotation; reuse of an already-rotated token revokes the ENTIRE token
                       family (tracked via token_family_id in Redis), forcing full re-authentication
```

---

## 9. Non-Functional Requirements

- **Performance:** tiered SLAs per §6 table (core CRUD <100ms p95, WebSocket <50ms, AI-backed <3s with immediate loading state, file upload <5s for ≤10MB).
- **Scalability:** target 5,000 concurrent hospital tenants, 100,000 active patient sessions; stateless Express cluster behind NGINX, PgBouncer transaction pooling, Postgres read replicas for analytics/search queries, Redis caching for hospital metadata and crowd status.
- **Availability:** 99.95% uptime SLA target, active-passive Postgres failover, Redis Sentinel replication.
- **Observability:** structured JSON logs (Pino), Prometheus metrics (`prom-client`), OpenTelemetry distributed tracing, Sentry error tracking (frontend + backend).

## 10. Security, Compliance & Scalability Implementation

### 10.1 OWASP Top 10 hardening
- **Injection:** parameterized SQL exclusively; `set_config($1,$2,true)` for RLS context — zero raw string interpolation anywhere in the codebase (enforced by lint rule + code review).
- **XSS/CSRF/CORS:** Helmet.js security headers, CORS origin allowlist locked to known frontend domains, HTML input sanitization on any rich-text field, SameSite cookies for refresh tokens.
- **Rate limiting:** `express-rate-limit` — 100 req/min per IP globally, 5 req/min on auth endpoints, dedicated (higher) limit tier for the emergency-trigger endpoint since throttling a real SOS request is unacceptable — instead, abuse on that endpoint is handled via CAPTCHA-on-repeat-failure, not hard rate limiting.

### 10.2 Identity & data encryption
- Argon2id password hashing, unique per-user salt.
- AES-256-GCM for `users.mfa_secret_encrypted` (§5.7).
- TLS 1.3 in transit everywhere; Ed25519 asymmetric signatures for billing receipts, verified via each tenant's `signing_public_key`.
- Client-side: JWT access token kept in memory (Zustand), **never** in `localStorage`, to reduce XSS token-theft blast radius; refresh token in an HttpOnly cookie only.

### 10.3 India DPDP Act 2023 compliance
- **Right to erasure:** implemented as soft anonymization (`patient_passports.is_anonymized = TRUE`, PII fields scrubbed) rather than hard deletion, since Indian clinical-establishment record-retention norms take precedence over raw delete-on-request — this policy is stated explicitly in the patient consent flow UI copy.
- **ABHA linkage:** `abha_id` field ships on `patient_passports` from day one even though the ABDM integration itself is a Future item (§18) — retrofitting the field into a live table later is far more expensive.
- **Consent ledger:** `notification_consents` gates SMS/email per TRAI; critical in-app/push notifications are unaffected by consent status.
- **Break-glass auditing:** every access to a patient passport outside a standing consent grant (e.g. `EMERGENCY_OVERRIDE`) writes an `audit_logs` row with actor, target, and reason metadata.

### 10.4 File upload security
Lab PDF uploads: MIME-type allowlist, magic-byte verification (not just file extension), ClamAV (or equivalent) scan before storage, 10MB size cap, storage in a non-executable S3 path with signed-URL retrieval only.

### 10.5 Scalability architecture
- Stateless Express instances behind NGINX/PM2 — any instance serves any request.
- PgBouncer transaction-mode pooling.
- Redis caching (hospital metadata, crowd status, sessions).
- Read/write splitting: analytics and hospital-search queries route to a Postgres read replica; all writes go to primary.

### 10.6 Environment & configuration management **[NEW]**

Since no source document specifies what actually goes into `environment.js`:

```
# .env.example (backend)
NODE_ENV=development
PORT=4000
DATABASE_URL=postgres://user:pass@localhost:5432/healthcareplus
DATABASE_READ_REPLICA_URL=
REDIS_URL=redis://localhost:6379
JWT_ACCESS_PRIVATE_KEY=      # RS256 private key, PEM, base64-encoded
JWT_ACCESS_PUBLIC_KEY=
JWT_ACCESS_TTL=15m
JWT_REFRESH_TTL=7d
MFA_ENCRYPTION_KEY=          # 32-byte hex, from Secrets Manager
GEMINI_API_KEY=
RAZORPAY_KEY_ID=
RAZORPAY_KEY_SECRET=
RAZORPAY_WEBHOOK_SECRET=
S3_BUCKET=
S3_REGION=
CLAMAV_HOST=
SENTRY_DSN=
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
SENDGRID_API_KEY=
```
Validated on boot via Zod (`config/environment.js`) — the process refuses to start if any required variable is missing or malformed, rather than failing on first request. Secrets (`JWT_*`, `MFA_ENCRYPTION_KEY`, payment/API keys) are pulled from a Secrets Manager (AWS Secrets Manager / GCP Secret Manager) in staging/production, `.env` is local-dev-only and gitignored. Keys are rotated on a documented quarterly schedule; `MFA_ENCRYPTION_KEY` rotation requires a re-encryption migration script since it's not simply swappable without touching stored ciphertext.

### 10.7 Demo / seed data plan **[NEW]**

`backend/seed/` contains an idempotent script producing: 3 demo tenants (geocoded to real coordinates in a target city for realistic PostGIS distance testing), department + doctor_affiliation rows per tenant, 1 super_admin, 2 hospital_admins, 6 doctors, 4 patients with populated passports (varied allergy/chronic-condition data to exercise the CDS engine), a handful of `ambulance_units` with live-looking coordinates, and sample appointments across every `appointment_status` value so every UI state (including `NO_SHOW`, `CANCELLED`) is visible without manually walking a full flow. Runnable via `npm run seed`, safe to re-run (upserts by known fixture IDs, doesn't duplicate).

---

## 11. Edge Cases & Failure Modes

| Scenario | Handling |
|---|---|
| Redis cluster unavailable during booking | Falls to `allocateTokenFallback` (Postgres `FOR UPDATE`); if Postgres is also degraded, booking endpoint returns 503 rather than issuing an unprotected token |
| Doctor deactivated mid-day with pending queue | Existing `CONFIRMED` tokens remain visible with a "doctor unavailable — contact reception" banner; no auto-cancellation, since only a human can safely triage that queue |
| Two lite-appointment requests in the same 5s Redis lock window | Second request receives `LITE_SLOT_BUSY`; frontend retries automatically once after a short delay |
| Patient revokes doctor consent mid-appointment | Revocation takes effect on next request, not retroactively — doctor's already-open passport view remains visible until the appointment reaches `COMPLETED` |
| Ambulance driver goes offline after accepting | No automatic re-broadcast (deliberate, to avoid patient confusion from repeated reassignment); flagged to ER staff via `emergency:{dispatchId}` room for manual escalation |
| Lab PDF fails ClamAV scan | Upload rejected before storage; lab tech sees an explicit "file failed security scan" error; `lab_requests.status` remains `PROCESSING` |
| Redis desync (counter out of step with DB) | Unique-violation on insert triggers `allocateTokenFallback` automatically — self-healing, no manual intervention needed |
| Payment webhook arrives twice | Idempotent on Razorpay `payment_id`; second delivery is a no-op |
| Patient's payment succeeds but webhook delivery is delayed | Appointment stays `PENDING_PAYMENT` for up to the 15-minute expiry window; frontend polls order status and shows "confirming payment…" rather than a false failure |
| Doctor tries to book at a hospital they're not affiliated with | Rejected at the DB trigger level (`check_doctor_affiliation`), independent of any application-layer bug |
| Super admin's `x-tenant-id` header points to a deactivated tenant | `enforceTenantContext` returns 404 before any tenant-scoped query runs |

## 12. Monitoring, Logging & Observability

- **Structured logs:** Pino, JSON format, correlation ID per request propagated through service/repository layers, aggregated via Grafana Loki.
- **Metrics:** `prom-client` exporting request latency histograms per route (to verify the §6 SLA tiers in production, not just in load tests), queue-depth gauges for BullMQ, Redis/Postgres connection pool utilization.
- **Tracing:** OpenTelemetry spans across the HTTP → Service → Repository → DB boundary, essential for diagnosing the specific "was it Redis or Postgres that was slow" question this architecture's fallback logic can otherwise obscure.
- **Error tracking:** Sentry SDK on both frontend (per-route-group error boundaries, §7.7) and backend (uncaught exceptions + `error.middleware.js` capture), with PII scrubbing rules configured so patient names/passport data never land in Sentry events.
- **Alerting:** PagerDuty/Opsgenie integration on: emergency-trigger 5xx rate >0 (page immediately, zero tolerance), booking endpoint p95 breach, Redis/Postgres connection exhaustion, ClamAV scan failures spiking (possible malware campaign).

---

## 13. Phase-by-Phase Implementation Plan

This is the sequential build order. Each phase states *why* it exists, what it unblocks, and exactly what "done" means. Backend and frontend tracks are interleaved so no screen is built before its API exists (per §6.1), while independent tracks (e.g. Emergency SOS backend + Design System frontend) can run in parallel once their own prerequisites are met.

Total program length: **16 weeks**, two coordinated tracks (Backend Engineer(s) / Frontend Engineer(s)), assuming a small team of 2–4 engineers. Solo builders should expect proportionally longer per phase but should **not** reorder phases — the dependency chain is real, not just a suggested pace.

---

### PHASE 0 — Foundations, Tooling & Design Tokens (Week 1)

**Objective:** Stand up the repositories, tooling, database, and design-token system so every subsequent phase has a working base to build on. Nothing product-facing ships this phase — it is entirely infrastructure.

**Features to implement:** none (infrastructure only).

**Step-by-step tasks:**
1. Initialize `backend/` as a Node 20+ ESM TypeScript-or-JS repository; configure Pino logging, Zod env validation (`environment.js`, §10.6), ESLint (including the custom "no raw `pool.query` outside `*.repository.js`" rule and "no bare `SET LOCAL`" rule).
2. Initialize `frontend/` with Vite + React 18; install Zustand, TanStack Query v5, Socket.io client, GSAP 3 + ScrollTrigger.
3. Provision PostgreSQL 16 (with PostGIS extension available) and Redis 7+ for local dev (Docker Compose) and staging.
4. Set up the migration tool (`node-pg-migrate` or Knex) and write migration files `0001`–`0015` matching §4.2's DDL blocks exactly, in order.
5. Run all migrations against a fresh local DB; confirm RLS is enabled on every listed table and policies exist.
6. Build the seed script (§10.7) and run it locally.
7. Configure `tokens.css` (§7.1) with the exact palette and radii values; set up Storybook (or an equivalent lightweight component playground) for the `ui/` primitive components.
8. Set up CI (GitHub Actions or equivalent): lint, typecheck, run migrations up/down/up, run seed, run unit test placeholder, axe-core placeholder.

**Folder & file structure:** as defined in §7.8 (frontend) and §7.9 (backend) — created empty/skeleton this phase, populated in later phases.

**Components to build:** `ui/` primitives skeleton only (Button, Card, Input, Modal, Badge) styled against tokens.css, no business logic.

**APIs required:** none.

**Database changes:** full schema from §4.2 deployed.

**UI/UX requirements:** design tokens match §7.1 exactly; primitive components pass the anti-boxy checklist (§7.1 principles 1–4).

**Validation & testing checklist:**
- [ ] `npm run migrate:up` then `npm run migrate:down` then `npm run migrate:up` succeeds cleanly (irreversibility check, §4.5).
- [ ] RLS policy existence verified via a query against `pg_policies` for all 11+1 tables.
- [ ] Seed script runs twice without error or duplication.
- [ ] CI pipeline green on a trivial PR.
- [ ] Storybook renders all primitive components with tokens applied.

**Dependencies:** none — this is the starting point.

**Expected outcome:** an empty but fully wired application skeleton — database schema live, CI green, design tokens in place, no features yet.

**Completion criteria:** a new engineer can clone both repos, run one Docker Compose command, and have a working local database with seed data and a Storybook of styled (if empty) primitive components, in under 30 minutes.

---

### PHASE 1 — Multi-Tenancy Middleware & Core Data Access (Week 2)

**Objective:** Make tenant isolation real and enforced at every layer before any feature that touches tenant-scoped data is built — this is deliberately not deferred, because retrofitting tenant isolation onto existing feature code is how leaks happen.

**Features to implement:** tenant context resolution, PgBouncer-safe query wrapper, base repository pattern.

**Step-by-step tasks:**
1. Implement `enforceTenantContext` middleware exactly as specified in §5.2.
2. Implement `withTenantContext` wrapper exactly as specified in §5.2; wire the ESLint rule requiring its use for any tenant-scoped repository call.
3. Build `tenantRepository` (`findActiveById`, tenant CRUD for super_admin onboarding).
4. Build the base repository pattern other modules will extend (parameterized query helpers, transaction helpers).
5. Write an automated **cross-tenant isolation test suite**: seed two tenants, attempt to read tenant A's data using tenant B's context, assert zero rows returned. This suite runs on every PR from this point forward.
6. Grant the application DB role INSERT/SELECT-only on `audit_logs` (no UPDATE/DELETE) at the database-role level.

**Folder & file structure:** `backend/src/middleware/tenant.middleware.js`, `backend/src/repository/withTenantContext.js`, `backend/src/modules/tenant/`.

**Components to build:** none (backend-only phase).

**APIs required:** `POST /api/v1/super-admin/tenants` (minimal, for seeding real tenants beyond the demo seed).

**Database changes:** none beyond Phase 0 (schema already includes RLS).

**UI/UX requirements:** none this phase.

**Validation & testing checklist:**
- [ ] Cross-tenant isolation suite passes and is wired into CI as a required check.
- [ ] `x-tenant-id` header is proven ineffective for any non-super_admin role (integration test).
- [ ] PgBouncer transaction-pooling load test (even a small K6 script) confirms no tenant-context bleed under concurrency.
- [ ] `audit_logs` role grants verified (attempt an UPDATE as the app role, expect a permission error).

**Dependencies:** Phase 0 complete (schema + tooling).

**Expected outcome:** a tenant-isolation foundation so solid that every later phase can simply assume "queries are automatically tenant-scoped, full stop."

**Completion criteria:** the cross-tenant isolation suite is green, required in CI, and the team has agreed the ESLint rule catches 100% of raw-query attempts (verified by deliberately writing a violating line and confirming lint fails).

---

### PHASE 2 — Authentication, Authorization & Healthcare Passport (Weeks 3-4)

**Objective:** Every user can register, log in securely, and patients get their Healthcare Passport — the identity foundation everything else (bookings, prescriptions, consent) depends on.

**Features to implement:** registration/login, Argon2id hashing, RS256 JWT rotation with refresh-family revocation, TOTP MFA (mandatory for admin roles), Healthcare Passport CRUD, ABHA ID field, consent grant/revoke, DPDP soft-anonymization.

**Step-by-step tasks:**
1. Build `auth` module: register, login, refresh, MFA setup/verify controllers/services/repositories.
2. Implement Argon2id hashing on registration; implement the `BYPASSRLS`/`SECURITY DEFINER` login lookup exception documented in §4.2's migration note.
3. Implement RS256 JWT issuance (15-min access, 7-day refresh) and the refresh-rotation-with-family-revocation logic (`token_family_id` tracked in Redis; reuse of a rotated token revokes the whole family).
4. Implement TOTP MFA setup/verify using `mfaCrypto.js` (§5.7); enforce mandatory MFA for `super_admin`/`hospital_admin` at the auth-service level (login flow branches to a session-token-then-MFA-verify path for these roles).
5. Build `passport` module: `patient_passports` CRUD, `passport_consent_grants` grant/revoke endpoints, DPDP anonymization endpoint (`is_anonymized = TRUE`, PII scrub, retains de-identified clinical record).
6. Add `abha_id` field to passport creation/update flow (field only, no ABDM integration).
7. Wire `notification_consents` creation on registration (default opt-in state per TRAI, patient can change immediately).

**Folder & file structure:** `backend/src/modules/auth/`, `backend/src/modules/passport/`; frontend: `pages/Auth/`, `store/useAuthStore.js`.

**Components to build (frontend, starts this phase per §6.1 ordering):** Auth & Role Onboarding Modal (screen 2, §7.5) — role selector pills, TOTP 6-digit auto-focus input; `useAuthStore` Zustand slice holding in-memory access token (never localStorage, §10.2).

**APIs required:** `/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/mfa/setup`, `/auth/mfa/verify`, `/passport/*`, `/passport/consent/grant`, `/passport/consent/revoke/:id`.

**Database changes:** none beyond Phase 0 schema (already includes all needed columns/tables).

**UI/UX requirements:** Auth modal per §7.5 screen 2 spec — soft rounded white card (`--r-lg`), teal active border, role pill buttons, MFA input auto-focuses on mount.

**Validation & testing checklist:**
- [ ] Argon2id verified (no plaintext password ever logged or stored).
- [ ] Refresh-token reuse test: replay an already-rotated token, confirm the entire family is revoked and the user is forced to re-authenticate.
- [ ] MFA enforced for `hospital_admin`/`super_admin` — login without MFA verification for these roles is rejected.
- [ ] `mfa_secret_encrypted` never appears in logs, error responses, or API payloads (grep CI check).
- [ ] Consent revoke takes effect on next request per the edge case in §11 (test: revoke mid-session, confirm already-open request context isn't retroactively broken, but a new request is denied).
- [ ] Anonymization endpoint verified to scrub PII while retaining a de-identified clinical record (not a hard delete).

**Dependencies:** Phase 1 (tenant middleware — auth routes are the one documented RLS exception, everything after login requires tenant context).

**Expected outcome:** every subsequent phase can authenticate a test user of any role and rely on `req.user`/`req.tenantId` being correctly populated.

**Completion criteria:** a QA engineer can register a patient, doctor, and hospital_admin account through the actual UI, log in as each (MFA-gated for admin), and view/edit a patient passport with a working consent grant/revoke cycle.

---

### PHASE 3 — Multi-Hospital Engine, Queue & Lite Appointments (Weeks 5-6)

**Objective:** Hospitals can be onboarded with departments and doctor affiliations, and patients can book a real, race-condition-free queue token — the core transactional loop of the product.

**Features to implement:** hospital/department/doctor-affiliation management, atomic Redis token allocation + Postgres fallback, Lite Appointment fractional tokens, idempotent booking, AI Crowd Status engine, hospital discovery (PostGIS nearby search).

**Step-by-step tasks:**
1. Build `tenant` module extensions: department CRUD, `doctor_affiliations` CRUD (consultation fee + lite fee), `tenant_settings` CRUD (branding, hours, `daily_token_capacity_per_doctor`).
2. Implement `check_doctor_affiliation` trigger verification (already in DDL from Phase 0 — write the integration test proving a cross-tenant booking attempt is rejected).
3. Build `QueueService.allocateRegularToken` / `allocateTokenFallback` / `allocateLiteToken` exactly as specified in §5.4.
4. Build the idempotency middleware (§5.3) and wire it onto `/appointments/book` and `/appointments/book-lite`.
5. Build `computeCrowdStatus` + Redis-cached `getTenantCrowdStatus` (§5.9).
6. Build `GET /hospitals/nearby` using `ST_Distance`/`ST_DWithin` against `tenants.location`.
7. Build Socket.io queue room (`queue:{tenantId}:{doctorId}`) and `authorizeRoomAccess` for the `queue` room type (§5.8).
8. Frontend: Hospital Workspace (screen 6), Doctor Booking Wizard with Lite toggle (screen 7), User Home Dashboard hospital grid + crowd badges (part of screen 3), Live Queue Ticker component subscribing to the socket room.

**Folder & file structure:** `backend/src/modules/queue/`, `backend/src/modules/tenant/` (extended); frontend `pages/HospitalExplorer/`, `components/queue/`.

**Components to build:** Nearby Hospitals card grid, Department→Doctor→Slot picker, Lite Appointment pill toggle, Live Queue Ticker (`{currentToken, yourToken, patientsAhead, estimatedWaitMinutes}`).

**APIs required:** `/hospitals/nearby`, `/appointments/book`, `/appointments/book-lite`, `/queue/live/:doctorId`, tenant/department/affiliation CRUD.

**Database changes:** none beyond Phase 0 schema.

**UI/UX requirements:** crowd status badge always shows color + text label together (§7.6 accessibility rule); Lite toggle only rendered client-side when the API confirms a `COMPLETED` appointment with that doctor in the last 90 days — never inferred client-side from stale cache.

**Validation & testing checklist:**
- [ ] Concurrent-booking load test (K6, simulated 2,000 req/sec) — zero duplicate token numbers.
- [ ] Lite Appointment concurrency test: 2 simultaneous lite requests for the same doctor/date resolve to distinct token numbers (0.1-increment retry verified).
- [ ] Idempotency-Key replay test: identical retried booking request returns the original appointment, not a duplicate.
- [ ] Doctor-affiliation trigger test: booking a doctor at a non-affiliated tenant is rejected with a clear error.
- [ ] Crowd status thresholds verified against the exact ratio bands (`<0.3` Low, `<0.7` Moderate, else High).
- [ ] Socket room authorization: a patient without an active appointment cannot join another doctor's queue room.

**Dependencies:** Phase 2 (auth/tenant context), Phase 1 (isolation).

**Expected outcome:** a patient can discover nearby hospitals sorted by distance with live crowd badges, book a real appointment (regular or lite) with a guaranteed-unique token, and watch their queue position update live.

**Completion criteria:** end-to-end demo — two patients book simultaneously against the same doctor/date and both receive correct, distinct tokens; the queue ticker updates live when a doctor advances the queue.

---

### PHASE 4 — AI Triage & Emergency SOS Dispatch (Weeks 7-8)

**Objective:** Ship the two AI-adjacent, safety-sensitive features — symptom triage and emergency ambulance dispatch — with their required guardrails and fallback paths built in from the start, not bolted on after a demo reveals a gap.

**Features to implement:** AI Symptom Triage (Gemini, department-only structured output, SSE streaming), PostGIS emergency dispatch, atomic driver acceptance lock, zero-candidate 108/102 escalation, authenticated Socket.io server with per-room authorization.

**Step-by-step tasks:**
1. Build `triage` module: `POST /triage/analyze`, Gemini client wrapper constrained to structured JSON output mapping to an existing `departments.name` for the requesting patient's nearby hospitals (§ v3.1 workflow: a suggestion for a department a given hospital doesn't have simply doesn't surface a "Book" shortcut there — no dead-end suggestions).
2. Implement the mandatory safety disclaimer rendering rule: every triage response ships with the fixed disclaimer and never blocks/hides the SOS button.
3. Build `emergency` module fully: `triggerEmergency`, `acceptDispatch`, zero-candidate immediate escalation, 15s driver-timeout escalation (§5.5).
4. Build `ambulance_units` CRUD + `is_available` online/offline toggle for drivers.
5. Stand up the authenticated Socket.io server (JWT handshake, §5.8) and `emergency:{dispatchId}` room.
6. Frontend: Emergency SOS hold button + confirmation bottom sheet (screen 3 element), Uber/Rapido-style live tracking map (screen 8), Ambulance Driver Console (screen 10) with Online/Available toggle and 15s accept sheet.
7. Implement the accessible SOS alternative (double-tap-and-confirm) per §7.6.
8. Implement the SOS-specific error-handling fallback (§7.7): "Call 108 directly" tel: link on network failure.

**Folder & file structure:** `backend/src/modules/triage/`, `backend/src/modules/emergency/`, `backend/src/services/socket.service.js`, `backend/src/services/socket.authorization.js`; frontend `pages/EmergencySOS/`, `components/emergency/`, driver console under `pages/DriverConsole/` (or a dedicated staff-console route group).

**Components to build:** AI Health Assistant banner with symptom input, Emergency SOS hold button + pulse animation (`animateCtaPulse`), live map tracker with dynamic ETA, driver's 15s dispatch bottom sheet with Accept/Decline.

**APIs required:** `/triage/analyze` (SSE), `/emergency/trigger`, `/emergency/accept/:id`, ambulance CRUD.

**Database changes:** none beyond Phase 0 schema.

**UI/UX requirements:** SOS state-machine copy exactly as specified in §8.1; cancellation UI hidden once `EN_ROUTE_PATIENT` or later per the deliberate design decision in §8.1.

**Validation & testing checklist:**
- [ ] Zero-candidate PostGIS search verified to trigger immediate `ESCALATED_108` without waiting for the 15s timer.
- [ ] Atomic driver-acceptance lock tested under concurrent driver acceptance calls — exactly one driver wins, others receive `ALREADY_ASSIGNED_OR_CLOSED`.
- [ ] Socket.io JWT handshake rejects unauthenticated connections; `authorizeRoomAccess` rejects a patient joining another patient's `emergency:{dispatchId}` room.
- [ ] Triage disclaimer renders on every response; SOS button never occluded or disabled by a triage UI state.
- [ ] Accessible SOS alternative (double-tap) verified with a screen reader.
- [ ] Network-failure fallback ("Call 108") verified by simulating an offline trigger call.

**Dependencies:** Phase 2 (auth/passport for triage's future personalization), Phase 3 (tenant/hospital data for triage department matching).

**Expected outcome:** a patient describing symptoms gets a safe, department-level AI suggestion; a patient triggering SOS gets either a real ambulance dispatch with live tracking or an immediate, visible escalation path — never a silent dead end.

**Completion criteria:** live demo of a zero-ambulance-nearby scenario correctly escalating within the UI, and a successful end-to-end dispatch (trigger → accept → live GPS updates → arrival) with two simulated driver clients racing for the same dispatch and only one winning.

---

### PHASE 5 — Clinical Workflows: CDS, Prescriptions, Lab, Pharmacy, Billing (Weeks 9-10)

**Objective:** Complete the clinical and financial loop — a doctor can safely prescribe, patients can fulfill prescriptions and lab orders, and every transaction produces a transparent, cryptographically verifiable receipt.

**Features to implement:** Hybrid CDS Engine, e-prescription generation, pharmacy order fulfillment + dose_logs reminders, lab request workflow + ClamAV-scanned PDF upload, itemized billing with Ed25519 signatures, payment gateway integration.

**Step-by-step tasks:**
1. Build `HybridCDSEngine.evaluatePrescription` exactly per §5.6, starting with pediatric-contraindication and allergy rules, extended with dosage-threshold and drug-interaction rules as a Phase 5 stretch task.
2. Build `POST /prescriptions/sign`: blocks on `REJECTED` CDS result, persists `cds_evaluation` JSONB audit trail, only allows signature on `APPROVED`.
3. Build `pharmacy` module: order state machine (§8.2), 48h auto-expiry BullMQ delayed job, `dose_logs` generation on `COMPLETED` (parse `dosage_per_day` formats: `"1-0-1"`, `"1-1-1-1"`, `"SOS"`).
4. Build `lab` module: request state machine (§8.3), ClamAV-scanned + magic-byte-checked PDF upload endpoint, in-app notification fan-out on `COMPLETED`.
5. Build `billing` module: `billing_receipts` + `receipt_items` creation on appointment/lab/pharmacy completion, Ed25519 signing using each tenant's `signing_public_key`, itemized receipt retrieval endpoint.
6. Build `payments` module per §5.11: Razorpay order creation, signature-verified webhook handler, idempotent-on-`payment_id` processing, 15-minute unpaid-appointment expiry job.
7. Frontend: Doctor Desk (screen 9) with CDS warning banners (blocking red for `REJECTED`, dismissible AI-explanation note for `APPROVED`), Pharmacy Fulfillment Console (screen 11) status stepper, Laboratory Console (screen 12) with PDF drop zone, dose_logs checkboxes on Personal Patient Dashboard (screen 4), billing receipt view in Hospital Workspace (screen 6).

**Folder & file structure:** `backend/src/modules/{triage(cds.service.js), pharmacy, lab, billing, payments}/`; frontend `pages/DoctorConsole/`, `pages/PharmacyConsole/`, `pages/LabConsole/`, `components/billing/`.

**Components to build:** e-Prescription generator with medicine line-item builder, CDS warning banner (blocking vs. advisory variants), pharmacy status stepper, ClamAV drop-zone uploader with progress + rejection states, dose checklist with tap-to-mark-taken, itemized receipt card with verification badge.

**APIs required:** `/prescriptions/sign`, `/pharmacy/orders/:id/confirm`, `/lab/reports/upload`, `/dose-logs/:id/taken`, `/payments/create-order`, `/payments/webhook`, billing retrieval endpoints.

**Database changes:** none beyond Phase 0 schema.

**UI/UX requirements:** `REJECTED` CDS banner is non-dismissible until the prescription is edited (§5.6); `APPROVED` AI summary is clearly labeled "AI explanation, not a safety check" so the UI itself reinforces which layer is authoritative.

**Validation & testing checklist:**
- [ ] Deterministic CDS rules unit-tested against known pediatric-contraindication and allergy test vectors before any AI layer sign-off.
- [ ] `dose_logs` generation verified against at least the three documented dosage-string formats.
- [ ] Pharmacy 48h auto-expiry BullMQ job verified (fast-forwarded in test).
- [ ] Lab upload pipeline: a corrupted/malicious test file is rejected before storage; `lab_requests.status` remains `PROCESSING`, not silently advanced.
- [ ] Billing receipt digital signature independently re-verifiable against the tenant's public key.
- [ ] Payment webhook replay test: duplicate delivery of the same `payment_id` is a no-op.
- [ ] Unpaid-appointment 15-minute expiry verified to free the token slot.

**Dependencies:** Phase 3 (appointments/queue), Phase 4 (not a hard blocker, but CDS conceptually pairs with the same AI-safety framing established there).

**Expected outcome:** the full clinical-to-billing loop works end to end: book → consult → prescribe (CDS-gated) → pharmacy fulfillment with reminders → lab order with secure report delivery → itemized, signed receipt.

**Completion criteria:** a single demo patient journey from booking through picking up medicine and viewing a verified receipt completes without any manual database intervention, and a deliberately-triggered CDS rejection correctly blocks prescription signing in the UI.

---

### PHASE 6 — Full Frontend Build-Out: Patient Portal Polish & Staff/Admin Consoles (Weeks 11-13)

**Objective:** Complete every remaining screen from the 14-screen map, apply the full design system and GSAP animation layer consistently, and reach cross-device responsive parity.

**Features to implement:** Landing page, Universal Healthcare Passport timeline UI, Hospital Admin Panel, Platform Super Admin Console, notification bell/inbox UI, full GSAP scroll-reveal + CTA pulse coverage, mobile bottom-sheet system, complete accessibility pass.

**Step-by-step tasks:**
1. Build Landing Page & Public Directory (screen 1) — hero, AI triage teaser, animated stat counters, hospital preview grid.
2. Build Universal Healthcare Passport screen (screen 5) — cross-hospital timeline cards, ABHA badge, consent grant/revoke toggle UI wired to Phase 2 endpoints.
3. Build Hospital Admin Panel (screen 13) — staff management, department/fee editing, queue token override (writes `audit_logs`, §5's admin-override edge case), lab/pharmacy pricing, ER monitor, revenue analytics from `billing_receipts`.
4. Build Platform Super Admin Console (screen 14) — tenant onboarding, network capacity analytics.
5. Build the persistent in-app notifications bell/inbox UI wired to `notifications` table + socket push.
6. Apply `initScrollReveals`/`animateCtaPulse` consistently across all patient-facing screens; verify `prefers-reduced-motion` short-circuit.
7. Build the mobile bottom-sheet component and touch-target accessibility wrapper (min 48px) used across search filters, booking, and SOS confirmation.
8. Full cross-device responsiveness audit (Chrome DevTools device emulation + at least 2 real devices per tier) against §7.4's three device tiers.
9. Full accessibility pass: axe-core CI gate, manual NVDA + VoiceOver pass on all 14 screens, keyboard-navigation pass (no mouse) on booking and SOS flows specifically.

**Folder & file structure:** remaining `frontend/src/pages/*` directories filled in per §7.8.

**Components to build:** all remaining primitives and page-level components not yet built in Phases 2–5; the notification bell dropdown; the admin data table with sticky headers and bulk selection.

**APIs required:** `/notifications`, `/notifications/:id/read`, `/admin/audit-logs`, `/admin/queue/override`, `/super-admin/tenants` (full CRUD, extending Phase 1's minimal version).

**Database changes:** none beyond Phase 0 schema.

**UI/UX requirements:** every screen conforms to §7.1's anti-boxy checklist and §7.6's accessibility requirements without exception; this phase is the enforcement point for both.

**Validation & testing checklist:**
- [ ] All 14 screens present and navigable from their respective role's entry point.
- [ ] axe-core CI check passes with zero critical/serious violations on patient-facing screens.
- [ ] Manual screen-reader pass completed and logged for all 14 screens.
- [ ] Cross-device audit checklist (3 tiers × representative real devices) signed off.
- [ ] Queue token override writes a verifiable `audit_logs` row every time.
- [ ] `prefers-reduced-motion` verified to disable scroll reveals and the SOS pulse.

**Dependencies:** Phases 2–5 (every screen here consumes an API built in an earlier phase — this phase adds zero new backend surface beyond notifications/admin CRUD).

**Expected outcome:** a feature-complete, fully responsive, accessible, light-theme, organic-UI frontend across all 14 screens and every role.

**Completion criteria:** a walkthrough of every role's primary journey (patient booking + SOS, doctor consultation + prescription, pharmacist fulfillment, lab tech upload, hospital admin override, super admin onboarding) is demoable end-to-end through the actual UI on mobile, tablet, and desktop viewports.

---

### PHASE 7 — Security Audit, Performance & Load Testing (Week 14)

**Objective:** Prove, not assume, that every security and performance claim made earlier in this document holds under adversarial and concurrent-load conditions before production traffic touches the system.

**Features to implement:** none (hardening/verification phase).

**Step-by-step tasks:**
1. Run the full cross-tenant RLS penetration test suite (extended beyond Phase 1's basic version) — attempt every documented attack angle: header spoofing, JWT tampering, RLS-bypass-via-`BYPASSRLS`-role misuse, direct DB access with a stolen app credential.
2. Run K6 load tests against the §6 SLA tiers specifically: booking endpoint at 2,000 req/sec, WebSocket queue updates at target concurrency, AI-triage endpoint under sustained load (verify graceful degradation, not cascading failure, if Gemini rate-limits).
3. Run a PgBouncer transaction-pooling concurrency test confirming tenant context never leaks under connection reuse.
4. Dependency/vulnerability scan (`npm audit`, Snyk or equivalent) on both repos; patch or document-and-accept every finding.
5. Verify `mfa_secret_encrypted`/PII never appear in logs, error responses, or Sentry events (automated grep + manual Sentry event sampling).
6. Verify refresh-token-family revocation, ClamAV scanning, and receipt-signature verification under direct attack simulation (replayed token, malicious file upload, tampered receipt payload).
7. Verify the payment webhook signature check rejects an unsigned/forged webhook payload.
8. Confirm automated database backup (hourly WAL archiving + daily full dump) with a **tested restore drill**, not just a cron job existing.

**Folder & file structure:** `backend/test/security/`, `backend/test/load/` (K6 scripts).

**Components to build:** none.

**APIs required:** none new — this phase tests everything already built.

**Database changes:** none.

**UI/UX requirements:** none.

**Validation & testing checklist:** (this phase *is* the checklist — see §17 for the consolidated master version, but at minimum):
- [ ] Cross-tenant RLS penetration suite: 100% of attempted attack vectors blocked.
- [ ] Booking endpoint sustains 2,000 req/sec with correct token uniqueness maintained throughout.
- [ ] Zero critical/high vulnerabilities in dependency scan, or each documented with an accepted-risk sign-off.
- [ ] Restore drill: a full database restore from backup completes and is verified against a known-good row count within the target RTO.
- [ ] Payment webhook forgery attempt rejected.

**Dependencies:** Phases 1–6 (everything must exist to be tested).

**Expected outcome:** documented, reproducible evidence — not just assertions in this document — that the system's security and performance claims hold.

**Completion criteria:** a signed-off security/performance report exists, every finding is either fixed or explicitly risk-accepted by a named decision-maker, and the master production-readiness checklist (§17) is fully checked.

---

### PHASE 8 — Production Deployment & Launch (Weeks 15-16)

**Objective:** Ship to production behind a real infrastructure topology, with monitoring live before the first real user touches the system.

**Features to implement:** none (deployment only).

**Step-by-step tasks:**
1. Provision production infrastructure matching §2.1's topology: Cloudflare WAF/DNS, NGINX with TLS auto-renewal, PM2-clustered (or containerized/K8s) Express nodes, PostgreSQL 16 primary + read replica, PgBouncer, Redis cluster.
2. Deploy via Docker Compose (small-scale) or Kubernetes (larger-scale) — CI/CD pipeline builds, tests, and deploys on merge to `main` with a manual production-promotion gate.
3. Wire Prometheus + Grafana Loki + Sentry + PagerDuty per §12, confirm alerts fire correctly against a synthetic incident before go-live (not just "the dashboard exists").
4. Run final DPDP compliance checklist sign-off (§10.3) with a named compliance owner.
5. Execute a soft launch with a single real (or design-partner) hospital tenant; monitor closely for 48–72 hours before opening broader onboarding.
6. Publish the production runbook: on-call rotation, incident response steps for each alert class in §12, rollback procedure.

**Folder & file structure:** `infra/` (Terraform/IaC if used), `docs/runbook.md`.

**Components to build:** none.

**APIs required:** none new.

**Database changes:** none — production database is initialized via the same migration set from Phase 0, run against production for the first time here.

**UI/UX requirements:** none new; production build verified pixel/behavior-identical to staging.

**Validation & testing checklist:**
- [ ] TLS certificates auto-renewing, verified with a forced near-expiry test if the provider supports it.
- [ ] All Phase 7 findings resolved or risk-accepted before go-live.
- [ ] Alerting verified against a synthetic incident (e.g. deliberately triggering a 5xx spike in staging-mirroring-production config).
- [ ] Runbook reviewed by every on-call engineer.
- [ ] Soft-launch tenant's real booking, SOS, and billing flows verified with real (not seed) data.

**Dependencies:** Phase 7 sign-off.

**Expected outcome:** healthcare+ is live in production, serving at least one real hospital tenant, with monitoring and an on-call process actually exercised, not just configured.

**Completion criteria:** 72 hours of stable production operation with the soft-launch tenant, zero unresolved P1/P2 incidents, and a go/no-go decision made explicitly by the team to proceed with broader tenant onboarding.

---

## 14. Testing Strategy

| Layer | Tooling | Scope |
|---|---|---|
| Unit | Jest | Backend services and utility functions (queue math, CDS rules, crypto helpers) in isolation |
| Integration | Supertest + test Postgres DB | Express route handlers against a real (test) database, including RLS behavior |
| Frontend component | React Testing Library | UI components and custom hooks in isolation |
| End-to-end | Playwright | Full patient appointment-booking journey, emergency SOS journey, doctor prescription-with-CDS-rejection journey |
| Load | K6 | Booking endpoint, WebSocket queue updates, AI-triage endpoint (§13 Phase 7) |
| Security | Custom pen-test suite + Snyk/npm audit | Cross-tenant isolation, auth/token attacks, upload attacks, webhook forgery |
| Accessibility | axe-core (CI) + manual NVDA/VoiceOver | All 14 screens (§7.6) |

## 15. Risks & Mitigation

| Risk | Severity | Impact | Mitigation |
|---|:---:|:---:|---|
| Multi-tenant data bleed | High | Critical | RLS on every tenant-scoped table, automated cross-tenant isolation tests on every PR, penetration test before launch (Phase 7) |
| Emergency GPS/network signal drop | High | High | Local dead-reckoning fallback on the driver client, immediate auto-escalation to 108/102 after zero-candidate or 15s driver timeout, "Call 108 directly" UI fallback on request failure |
| AI triage hallucination/misdiagnosis | Medium | High | Output constrained strictly to department-name matching against real hospital data; mandatory disclaimer on every response; SOS button never gated behind or hidden by triage UI |
| CDS engine wrongly approving a dangerous prescription | Medium | Critical | Deterministic rule engine is the sole blocking authority; AI is advisory-only and cannot suppress or originate a warning; rule engine unit-tested against known contraindication vectors before any release |
| Redis outage during peak booking | Medium | Medium | Automatic Postgres `FOR UPDATE` fallback, self-healing on unique-violation, load-tested in Phase 7 |
| Payment webhook delivery failure/delay | Medium | Medium | 15-minute unpaid-appointment expiry frees the slot; frontend shows "confirming payment" rather than false failure |
| Team reordering phases under deadline pressure | Medium | High | This document's phase dependencies are explicit and load-bearing (§13); PM sign-off required to skip/reorder any phase, with the specific dependency being knowingly waived documented in writing |

## 16. Monitoring & Observability — Tooling Summary

(Consolidated from §12 for quick reference.) Pino (logs) → Grafana Loki (aggregation). `prom-client` (metrics) → Prometheus → Grafana dashboards. OpenTelemetry (tracing). Sentry (errors, both frontend and backend, PII-scrubbed). PagerDuty/Opsgenie (alerting on the specific triggers listed in §12).

## 17. Future Enhancements (Explicitly Post-Launch)

1. **Telemedicine video consultations** — WebRTC peer-to-peer video rooms for remote appointments.
2. **IoT vitals streaming** — Bluetooth LE integration with smart watches/pulse oximeters streaming live vitals directly into the Healthcare Passport.
3. **Full ABDM/FHIR interoperability** — the `abha_id` field ships in v1 (§10.3); the actual HL7 FHIR data-exchange integration with India's ABDM network is a dedicated future project.
4. **Dark mode** — deliberately out of scope for v1 per the design-system resolution in §0.2/§7.1; revisit only if user research specifically requests it.
5. **Doctor rating & patient feedback module.**
6. **Advanced hospital explorer search/filter grid** (beyond the nearest-4 grid shipped in v1).
7. **International compliance** — full HIPAA/GDPR program for future US/EU tenants, architected for (data-residency-aware schema design) but not implemented in v1.
8. **Self-service patient refunds** — v1 ships admin-initiated refunds only (§5.11); a patient-facing refund request flow is a natural Phase 9+ addition.

---

## 18. Master Production Readiness Checklist (Consolidated)

Every item below must be checked before broader tenant onboarding proceeds past the Phase 8 soft launch.

**Database & multi-tenancy**
- [ ] All 15+ PostgreSQL tables deployed with foreign keys, GIST spatial indexes, and RLS policies.
- [ ] RLS enabled and policy-tested on all 11 tenant-scoped tables plus the `emergency_dispatches` special-case policy.
- [ ] `users` table RLS bypass path for pre-auth login verified not to leak cross-tenant emails.
- [ ] Automated cross-tenant isolation test suite passing and required in CI.
- [ ] `withTenantContext` wrapper enforced across all repository calls via lint rule.
- [ ] PgBouncer transaction-pooling load test confirms no tenant-context bleed under concurrency.

**Auth & security**
- [ ] Argon2id password hashing verified.
- [ ] RS256 JWT rotation with Redis refresh-token-family revocation functional; reuse-detection tested.
- [ ] MFA (TOTP) enforced for `hospital_admin` and `super_admin`.
- [ ] `mfa_secret_encrypted` verified never appears in logs, error messages, or API responses.
- [ ] Doctor-affiliation trigger verified to reject booking a doctor at a non-affiliated tenant.
- [ ] Socket.io JWT handshake + `authorizeRoomAccess` verified to reject unauthorized room joins.
- [ ] Lab upload pipeline scans files via ClamAV and verifies magic bytes before attaching to Passport.
- [ ] Payment webhook signature verification rejects forged payloads.
- [ ] Rate limiting active on auth and abuse-prone endpoints; emergency-trigger endpoint uses CAPTCHA-on-abuse, not hard throttling.
- [ ] CORS locked to trusted domains; TLS certificates auto-renewing.

**Core transactional correctness**
- [ ] Idempotency-Key replay test: identical retried booking request returns the original appointment, not a duplicate.
- [ ] Atomic Redis queue counter + Postgres fallback verified under simulated 2,000 req/sec load.
- [ ] Lite Appointment concurrent-booking test resolves to distinct token numbers.
- [ ] Emergency driver-acceptance atomic lock tested under concurrent acceptance calls.
- [ ] Zero-candidate PostGIS search verified to trigger immediate 108/102 escalation.
- [ ] Pharmacy 48h auto-expiry BullMQ job verified.
- [ ] `dose_logs` generation verified against at least 3 real-world dosage string formats.
- [ ] Unpaid-appointment 15-minute expiry verified to free the token slot.
- [ ] Billing receipt digital signatures independently re-verifiable.

**Clinical safety**
- [ ] Deterministic CDS rule engine unit-tested against pediatric-contraindication and drug-allergy test vectors before any AI-layer sign-off.
- [ ] CDS `REJECTED` result verified non-dismissible in the doctor UI without editing the prescription.

**Compliance**
- [ ] DPDP Act 2023 right-to-erasure soft-anonymization verified.
- [ ] ABHA ID field functional (integration itself deferred to Future Enhancements).
- [ ] Notification consent ledger enforced per TRAI; critical notifications unaffected by consent status.
- [ ] Break-glass passport access writes an `audit_logs` row every time.

**Frontend & accessibility**
- [ ] All 14 screens present, responsive across the three device tiers, and light-theme/anti-boxy compliant.
- [ ] axe-core CI check passes with zero critical/serious violations on patient-facing screens.
- [ ] Manual screen-reader pass completed and logged.
- [ ] `prefers-reduced-motion` correctly disables scroll reveals and the SOS pulse.
- [ ] Accessible SOS alternative (double-tap-confirm) verified.

**Infrastructure & operations**
- [ ] Redis eviction/persistence configured.
- [ ] JWT secrets, DB credentials, and all third-party API keys stored in a Secrets Manager, not `.env` in production.
- [ ] Automated hourly WAL archiving + daily full DB backup, with a **tested restore drill**.
- [ ] Monitoring stack (Pino/Loki, Prometheus/Grafana, OpenTelemetry, Sentry, PagerDuty) live and alert-tested before go-live.
- [ ] Production runbook reviewed by every on-call engineer.
- [ ] 72-hour soft-launch stability window completed with zero unresolved P1/P2 incidents.

---

*End of document. This is the single source of truth for healthcare+ from Phase 0 through production launch. Where any future planning note conflicts with this document, this document governs unless explicitly superseded in writing with its own change-log entry, following the same provenance-and-resolution pattern used in §0.*
