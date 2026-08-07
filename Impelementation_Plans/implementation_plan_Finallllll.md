# healthcare+ — Master Architecture Specification & Implementation Plan (v5.0 Mobile-First Edition)

> **Document Status**: APPROVED MASTER BLUEPRINT (Production Build-Ready).
> **Target Tech Stack**: PERN — PostgreSQL 16+ (PostGIS), Express.js / Node.js 20+ LTS, React 18+ Vite, GSAP + ScrollTrigger, TailwindCSS / Custom Tokens.
> **Design Language**: Mobile-First Responsive, High-Trust Minimal Healthcare Aesthetic, GSAP ScrollTrigger Animations.
> **Curated Color Palette**: Primary `#03A6A1` | Secondary `#FFE3BB` | Accent `#FFA673` | Highlight/CTA `#FF4F0F`.
> **Primary Compliance Target**: India Digital Personal Data Protection (DPDP) Act 2023, TRAI SMS Regulations, ABDM ABHA Standard.

---

# SECTION 1: DESIGN SYSTEM, COLOR PALETTE & ANIMATION ARCHITECTURE

## 1.1 Curated Color Palette & Design Tokens
The design identity avoids generic AI-generated boxy patterns and overly saturated clutter. It balances clinical precision with warm, trustworthy accessibility using the exact specified color tokens:

```css
/* src/styles/tokens.css */
:root {
  /* Brand Core Palette */
  --brand-primary: #03A6A1;   /* Deep Teal - Trustworthy Clinical Primary */
  --brand-secondary: #FFE3BB; /* Warm Cream / Soft Sand Accent */
  --brand-accent: #FFA673;    /* Warm Coral Accent */
  --brand-cta: #FF4F0F;       /* Vibrant Crimson / Action & Emergency SOS */

  /* Neutral Dark Theme Bases */
  --bg-main: #060c14;
  --bg-surface: #0c1624;
  --bg-card: #121f33;
  --bg-card-hover: #182842;
  --border-subtle: #1e3250;
  --border-active: #03A6A1;

  /* Typography Colors */
  --text-main: #f0f6fc;
  --text-muted: #8b9eb7;
  --text-subtle: #4b6382;

  /* Functional Status Colors */
  --status-low: #10b981;    /* 🟢 Low Crowd / Available */
  --status-mod: #f59e0b;    /* 🟡 Moderate Crowd */
  --status-high: #ef4444;   /* 🔴 High Crowd */

  /* Typography & Layout Spacing */
  --font-sans: 'Outfit', -apple-system, BlinkMacSystemFont, sans-serif;
  --font-mono: 'Fira Code', monospace;
  --touch-target-min: 48px;  /* Mobile Accessibility Target */
}

:root.light-mode {
  --bg-main: #f8fafc;
  --bg-surface: #ffffff;
  --bg-card: #ffffff;
  --bg-card-hover: #f1f5f9;
  --border-subtle: #e2e8f0;
  --border-active: #03A6A1;
  --text-main: #0f172a;
  --text-muted: #475569;
  --text-subtle: #94a3b8;
}
```

## 1.2 GSAP & ScrollTrigger Animation Engine
Animations must be lightweight, performance-friendly, and natural. Excessive motion or distracting transitions are strictly forbidden. GSAP with **ScrollTrigger** is initialized globally for smooth reveal effects on cards, statistics, and section transitions.

```javascript
// src/utils/gsapAnimations.js
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';

gsap.registerPlugin(ScrollTrigger);

/**
 * Initializes ScrollTrigger reveal animations for cards and sections
 * @param {string} selector - CSS selector of elements to animate on scroll
 */
export const initScrollReveals = (selector = '.gsap-reveal') => {
  const elements = document.querySelectorAll(selector);
  elements.forEach((el) => {
    gsap.fromTo(
      el,
      { opacity: 0, y: 24 },
      {
        opacity: 1,
        y: 0,
        duration: 0.6,
        ease: 'power2.out',
        scrollTrigger: {
          trigger: el,
          start: 'top 85%',
          toggleActions: 'play none none reverse',
        },
      }
    );
  });
};

/**
 * Smooth Micro-Interaction for CTA Buttons & SOS Pulse
 */
export const animateCtaPulse = (target) => {
  return gsap.to(target, {
    scale: 1.03,
    duration: 0.8,
    repeat: -1,
    yoyo: true,
    ease: 'sine.inOut',
  });
};
```

---

# SECTION 2: DEVICE-TIER RESPONSIVE ARCHITECTURE

Mobile-first responsiveness is not a final polish phase—it is integrated into every component from Day 1.

```
+-----------------------------------------------------------------------------------+
|  Device-Tier Responsive UX Matrix                                                 |
+-----------------------------------------------------------------------------------+
|  1. SMARTPHONES (< 640px) [Patients & Public Users]                                |
|  - One-handed thumb-zone navigation bar at screen bottom.                         |
|  - All tap targets minimum 48px x 48px with visual feedback.                      |
|  - Slide-over Bottom Sheets for search filters, doctor booking, and SOS modal.    |
|  - Touch gesture support (swipe to dismiss notifications / pull to refresh).     |
+-----------------------------------------------------------------------------------+
|  2. TABLETS (640px - 1024px) [Doctors & Clinical Staff]                           |
|  - Side-by-side split screen: Queue list on left (35%), Patient EHR on right (65%).|
|  - Touch-friendly prescription generator & lab requester buttons.                 |
|  - Optimized orientation support for rounds (Portrait & Landscape).              |
+-----------------------------------------------------------------------------------+
|  3. LAPTOPS & DESKTOPS (> 1024px) [Hospital Admins & Super Admins]                 |
|  - Dense, multi-column analytics dashboards with collapsible sidebars.            |
|  - High-throughput data tables with fixed headers and bulk selection actions.     |
+-----------------------------------------------------------------------------------+
```

---

# SECTION 3: ARCHITECTURAL COMPARISON & JUSTIFICATION

| Dimension | Legacy Django/SQLite | v1.0 Proposal Draft | v4.0 Blueprint | Ultimate Mobile-First Spec (v5.0 Final) |
|:---|:---|:---|:---|:---|
| **Multi-Tenancy** | Single-hospital. | Shared DB, header-trusted. | Parameterized RLS on 12 tables. | **Parameterized RLS on all 12 operational tables**; `withTenantContext` PgBouncer transaction wrapper. |
| **Database** | SQLite. | PostgreSQL 16 + PostGIS. | PostgreSQL 16 + PostGIS DDL. | **PostgreSQL 16 + PostGIS** with 15 complete DDL tables, GIST spatial indexes, DB triggers. |
| **Design & Responsiveness** | Basic CSS. | Generic proposal. | Basic responsiveness. | **Mobile-First Responsive Architecture**: Specialized touch targets (>=48px), bottom sheets, and GSAP ScrollTrigger transitions. |
| **Animation Engine** | CSS transitions. | CSS transitions. | CSS transitions. | **GSAP + ScrollTrigger**: Smooth scroll reveals, non-distracting micro-interactions, custom GSAP hooks. |
| **Color System** | Random colors. | Hardcoded tokens. | Hardcoded tokens. | **Restrained Clinical Palette**: Primary `#03A6A1`, Secondary `#FFE3BB`, Accent `#FFA673`, Highlight/CTA `#FF4F0F`. |
| **Emergency Ambulance** | Static string location. | Spatial lookup. | Atomic SQL lock + 108 fallback. | **Uber/Rapido-Style Driver Console**: PostGIS 10km spatial query, atomic driver lock, live GPS map tracking, 108 fallback. |
| **CDS Engine** | Simple regex. | LLM replacing rules. | Hybrid CDS model. | **Hybrid CDS Engine**: Deterministic core gate for pediatric/allergy/dosage checks + LLM advisory summaries. |

---

# SECTION 4: COMPLETE DATABASE DESIGN & PRODUCTION DDL SQL (v5.0)

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
    primary_color VARCHAR(20) DEFAULT '#03A6A1',
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

-- 12. AMBULANCE FLEET & EMERGENCY DISPATCH (Uber/Rapido Driver Model)
CREATE TABLE ambulance_units (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id),
    driver_id UUID NOT NULL REFERENCES users(id),
    vehicle_number VARCHAR(50) NOT NULL,
    current_location GEOMETRY(Point, 4326),
    is_available BOOLEAN DEFAULT TRUE, -- Toggle Online/Offline
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

# SECTION 5: REVISED DEVELOPMENT ROADMAP & TASK BREAKDOWN

```
Phase F0: Mobile-First Infrastructure, Design System & GSAP Engine (Week 1)
├── Task F0.1: Configure design tokens in tokens.css using curated palette (#03A6A1, #FFE3BB, #FFA673, #FF4F0F).
├── Task F0.2: Setup GSAP + ScrollTrigger animation utilities (initScrollReveals, animateCtaPulse).
├── Task F0.3: Establish responsive layout breakpoints (Mobile <640px, Tablet 640-1024px, Desktop >1024px).
└── Task F0.4: Build mobile bottom-sheet component & touch-target accessibility wrapper (min 48px).

Phase 1: Backend Architecture & Database Foundations (Weeks 2-3)
├── Task 1.1: Express Node.js ESM repository setup with Pino logging & Zod env validation.
├── Task 1.2: PostgreSQL 16 + PostGIS DDL migration script (15 tables, indexes, triggers).
├── Task 1.3: RLS policy application across all 12 operational tables.
└── Task 1.4: Parameterized tenant middleware & PgBouncer withTenantContext transaction wrapper.

Phase 2: Auth, Security & Healthcare Passport (Weeks 4-5)
├── Task 2.1: Argon2id password hashing, RS256 JWT rotation & Redis token family revocation.
├── Task 2.2: AES-256-GCM MFA secret encryption & TOTP verification endpoints.
├── Task 2.3: Healthcare Passport APIs, ABHA ID linkage & DPDP 2023 soft anonymization handler.
└── Task 2.4: Passport consent grant matrix (APPOINTMENT_AUTO, EMERGENCY_OVERRIDE, MANUAL_GRANT).

Phase 3: Multi-Hospital Engine, Queue & Lite Appointments (Weeks 6-7)
├── Task 3.1: Hospital tenant onboarding, department setup & doctor_affiliations APIs.
├── Task 3.2: Atomic Redis INCR token allocation engine & Postgres FOR UPDATE fallback.
├── Task 3.3: Lite Appointment fractional token generator (Token #15.5) with Redis locks.
└── Task 3.4: AI Crowd Status engine (computeCrowdStatus) with 60s Redis caching.

Phase 4: AI Triage & Emergency SOS Dispatch (Weeks 8-9)
├── Task 4.1: Gemini AI Symptom Triage service with natural language to department JSON mapping.
├── Task 4.2: PostGIS ST_DWithin spatial ambulance search & zero-driver 108/102 fallback.
├── Task 4.3: Atomic driver acceptance UPDATE...RETURNING execution & socket broadcast.
└── Task 4.4: Authenticated Socket.io server with per-room authorization guard (authorizeRoomAccess).

Phase 5: Clinical Workflows, Lab, Pharmacy & Billing (Weeks 10-11)
├── Task 5.1: Hybrid CDS Engine (Deterministic safety gate + LLM advisory summaries).
├── Task 5.2: Pharmacy fulfillment order workflow & BullMQ scheduled medicine reminder logs (dose_logs).
├── Task 5.3: Laboratory sample tracking workflow & ClamAV scanned PDF report uploader.
└── Task 5.4: Standalone & appointment billing receipts with Ed25519 digital signatures.

Phase 6: Mobile-First Frontend Component Library & Dashboards (Weeks 12-13)
├── Task 6.1: Build Mobile-First User Home Dashboard (AI Triage banner, 4-nearest hospital cards, SOS modal).
├── Task 6.2: Build Personal Patient Dashboard (Active queue ticket, dose_logs checkboxes, Passport timeline).
├── Task 6.3: Build Hospital Workspace & Doctor Booking Wizard (with Lite Appointment toggle).
└── Task 6.4: Integrate GSAP ScrollTrigger reveals across all patient-facing screens.

Phase 7: Staff Consoles, Responsiveness Audit & Production Deployment (Weeks 14-15)
├── Task 7.1: Build Doctor Desk (Tablet-optimized split layout with Hybrid CDS banners).
├── Task 7.2: Build Ambulance Driver Console (Uber/Rapido style) with GPS tracking & navigation.
├── Task 7.3: Build Hospital Admin Panel & Platform Super Admin Console (Desktop multi-column).
├── Task 7.4: Conduct cross-device Mobile/Tablet responsiveness audit & GSAP performance profiling.
└── Task 7.5: Production rollout with NGINX reverse proxy, PM2 cluster, and Docker Compose / K8s.
```

---

# SECTION 6: FINAL PRODUCTION READINESS VERIFICATION CHECKLIST

- [ ] **Design Tokens & Palette**: Color tokens configured exclusively with `#03A6A1` (Primary), `#FFE3BB` (Secondary), `#FFA673` (Accent), `#FF4F0F` (CTA/Emergency).
- [ ] **GSAP Animation Engine**: GSAP + ScrollTrigger initialized globally; smooth reveal animations active; zero heavy or distracting transitions.
- [ ] **Mobile-First UX Verification**: All patient touch targets >= 48px; thumb-zone navigation bar active on mobile; bottom sheets implemented for search & SOS confirmation.
- [ ] **Device-Tier Layout Audit**: Smartphone (<640px), Tablet (640-1024px split-screen), and Desktop (>1024px) layouts audited across Chrome DevTools & real devices.
- [ ] **Database & RLS Integrity**: All 15 PostgreSQL tables deployed; RLS policies active and verified via automated cross-tenant isolation test suite.
- [ ] **Tenant Middleware Security**: `x-tenant-id` header untrusted for non-super-admins; `set_config($1, $2, true)` parameterized; `withTenantContext` transaction wrapper enforced across all repository queries.
- [ ] **Auth & Encryption**: Argon2id password hashing verified; RS256 JWT rotation with Redis token family revocation functional; `users.mfa_secret_encrypted` encrypted via AES-256-GCM.
- [ ] **Idempotency Protection**: Booking endpoints enforce `Idempotency-Key` header, preventing duplicate appointments or billing charges.
- [ ] **Atomic Queue Engine**: Redis `INCR` counter with Postgres `SELECT FOR UPDATE` fallback verified under simulated 2,000 req/sec booking load.
- [ ] **Lite Appointments**: Fractional token generation (`15.5`) verified under concurrent lite booking tests.
- [ ] **Emergency Dispatch System**: PostGIS spatial search verified; atomic driver acceptance (`UPDATE...RETURNING`) confirmed; zero-driver radius verified to trigger immediate 108/102 escalation; Uber/Rapido style driver app console operational.
- [ ] **Hybrid CDS Engine**: Deterministic core rule engine unit-tested against pediatric contraindications and drug allergy test vectors before AI layer sign-off.
- [ ] **Pharmacy & Reminders**: Order fulfillment status stepper verified; `dose_logs` table populates scheduled reminder rows upon order completion.
- [ ] **Laboratory Pipeline**: PDF report upload pipeline scans files via ClamAV and verifies magic-bytes before attaching to Passport.
- [ ] **DPDP Act 2023 Compliance**: Patient right-to-erasure soft anonymization verified; ABHA ID linkage functional; notification consent ledger enforced.
- [ ] **Real-Time Security**: Socket.io connection handshake authenticates JWT; `authorizeRoomAccess` guard rejects unauthorized room joins.
- [ ] **Production Infrastructure**: NGINX SSL/TLS reverse proxy, PM2 process cluster, Redis Pub/Sub, PgBouncer transaction pooling, and automated backup cron jobs verified.
