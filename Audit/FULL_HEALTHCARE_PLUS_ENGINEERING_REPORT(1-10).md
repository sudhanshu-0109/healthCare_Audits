# Healthcare+ — Complete Software Engineering Report (Production-Ready Rebuild)

Repository: sudhanshu-0109/healthcare-  
Date: 2026-08-04  
Prepared by: Senior Software Architect (assistant)  
Stack: React (Vite) | Node.js (Express) | PostgreSQL | Prisma | JWT + Refresh Tokens | Tailwind CSS | Framer Motion | Axios

---

## Preface — Explicit Assumptions
- Single monolith backend (Node + Express) with Prisma ORM and PostgreSQL for initial implementation. Microservices considered later.
- Frontend: React SPA built with Vite, deployed to CDN.
- PHI (Personal Health Information) will be stored — HIPAA/GDPR-style controls required.
- Third-party integrations: payments (Stripe/Razorpay), email (SES/SendGrid), SMS (Twilio/MSG91), object storage (S3-compatible).
- Small cross-functional team (2–6 engineers) with QA & DevOps support.
- Goal: production-ready SaaS ready for pilot deployments and investor due diligence.

---

# Table of Contents
1. Executive Summary  
2. Product Overview  
3. Functional Requirements  
4. Non-Functional Requirements  
5. Complete User Flow  
6. Role Based Flow  
7. Information Architecture  
8. Complete Screen List  
9. Database Design  
10. Prisma Schema Planning  
11. API Design  
12. Folder Structure  
13. Frontend Architecture  
14. Backend Architecture  
15. Authentication Flow  
16. Security Architecture  
17. UI/UX Design System  
18. Development Roadmap (25 Phases)  
19. Testing Strategy  
20. Deployment Architecture  
21. Future Roadmap  
22. Risks & Mitigation  
23. Coding Standards  
24. Complete Development Flow  
25. Final Engineering Checklist  
Appendices (A–F)

---

# 1. Executive Summary
- Vision: Build a Transparent Healthcare Operating System (THOS) that orchestrates hospital workflows with full auditability, security, and AI-driven operational intelligence.
- Purpose: Reduce billing fraud, shorten waiting times, improve coordination and adherence, and provide patients transparent control of their health interactions.
- Target users: Hospitals/clinics (admins), doctors, lab/pharmacy staff, patients, ambulance partners.
- Problem: Fragmented hospital systems, manual workflows, billing opacity, slow emergency response.
- Solution: A secure SaaS platform with RBAC, immutable audit logs, reliable payments, secure file storage, and AI microservices for predictions and anomaly detection.
- Why rebuild: The existing repo contains fatal security and architecture issues (plain-text passwords, open endpoints, mocked payments, committed SQLite/venv). A rebuild enforces secure, maintainable patterns.
- Expected outcome: Pilot-ready, secure, scalable product with investor-ready documentation and compliance posture.

---

# 2. Product Overview
- Product: THOS — unified cloud-hosted platform for appointment booking, queue management, lab workflows, prescriptions, billing, emergency dispatch, and notifications.
- Goals: Security-by-design, auditability, automation, measurable AI improvements.
- Business objectives: pilot → hospital contracts → SaaS subscriptions + per-transaction fees; downstream integrations with HIS/EMR vendors.
- User personas:
  - Super Admin (platform-level)
  - Hospital Admin (finance/ops)
  - Doctor (clinical workflows)
  - Lab Staff (test processing)
  - Pharmacist (dispensing)
  - Patient (consumer)
  - Ambulance Partner (emergency)
- Stakeholders: Hospital leadership, clinicians, patients, investors, legal/regulatory.
- Core use cases: appointment booking, queue/visit management, lab ordering and report upload, digital prescriptions and reminders, billing + payments, emergency request/dispatch, notifications.

---

# 3. Functional Requirements
For each module we specify purpose, features, inputs, outputs, validations, DB impact, API, UI screens, edge cases, permissions, dependencies, scalability.

Note: Each feature should have an acceptance test and API contract before implementation.

## 3.1 Authentication & Identity
- Purpose: Secure identity & session handling.
- Features: registration (email OTP), email verification, login, JWT access + refresh, logout, forgot/reset, device/session management, role assignment.
- Inputs: name, email, password, role, OTP.
- Outputs: access token, refresh token, user DTO.
- Validations: password strength (≥8 char, complexity), email format, unique email, OTP expiry.
- DB: users, refresh_tokens, sessions, audit_logs.
- API: POST /auth/register, /auth/verify-email, /auth/login, /auth/refresh, /auth/logout, /auth/forgot-password, /auth/reset-password.
- UI: sign-up, verify, login, forgot/reset screens.
- Edge: duplicated emails, expired OTPs, rate-limit abuse.
- Permissions: public for register/login; authenticated for logout/refresh.
- Dependencies: email/SMS provider, rate limiter.
- Scalability: stateless JWT, refresh tokens stored hashed in Redis/Postgres.

## 3.2 Users & Roles
- Purpose: manage user accounts & RBAC.
- Features: user CRUD (admin), profile update, doctor directory, search.
- Inputs/Outputs: standard user DTO.
- Validations: email uniqueness, specialty allowed for doctors.
- DB: users table (role enum).
- API: /users endpoints.
- UI: admin user management, doctor directory, profile pages.
- Permissions: admin-only for list/update of other users; self for profile update.

## 3.3 Appointments & Queue Management
- Purpose: book & manage appointments, session limits, auto-queue.
- Features: doctor search, availability, booking, reschedule, cancel, real-time queue updates, queue auto-assignment, no-show handling.
- Validations: no double booking, slot availability, not past date.
- DB: appointments, appointment_slots (optional), queue state.
- API: GET/POST/PATCH appointments, POST /appointments/:id/reschedule.
- UI: booking wizard, calendar, doctor/patient dashboards.
- Edge: concurrent booking race (use DB transactions & SELECT FOR UPDATE).
- Permissions: patient create, doctor/admin update statuses.
- Scalability: availability caching, partitioning by doctor.

## 3.4 Prescriptions & Reminders
- Purpose: capture prescriptions, schedule reminders, track adherence.
- Features: nested meds, dosage & schedule, reminders, missed-dose tracking, adherence analytics.
- DB: prescriptions, prescription_items, reminder_logs.
- API: POST /prescriptions, GET /prescriptions?patient_id.
- UI: clinician form, patient view, reminder settings.
- Permissions: doctor create; patient read own.

## 3.5 Lab Workflow & Reports
- Purpose: order tests, fixed pricing, file uploads (PDF), report distribution.
- Features: price locking, admin approval for changes, order assignment, report upload (S3), file validation & scan.
- DB: lab_pricing, lab_tests, lab_files.
- API: POST /lab-tests, GET /lab-tests, POST /lab-tests/:id/upload-report.
- UI: lab order form, lab staff dashboard, patient report viewer.
- Edge: large files, upload interruption, virus detection failure.
- Permissions: doctor order, lab staff upload.

## 3.6 Billing & Payments
- Purpose: itemized billing, payment processing, receipts, refunds.
- Features: create bills, link orders to bills, payment order creation, webhook verification, refunds & disputes.
- DB: billing, payment_orders, payment_transactions.
- API: POST /billing, GET /billing?patient_id, POST /payments/create-order, POST /payments/verify, POST /payments/webhook.
- Validation: price consistency, idempotency keys.
- Permissions: admin create, patient pay own.
- Security: always verify provider signature; no mock bypass.

## 3.7 Pharmacy Orders
- Purpose: fulfill prescriptions via pharmacy.
- Features: prescription linkage, nested items, pickup/delivery statuses, inventory checks.
- DB: pharmacy_orders, pharmacy_items.
- API: POST /pharmacy-orders, PATCH /pharmacy-orders/:id/pickup.
- Permissions: patient create, pharmacist update.

## 3.8 Emergency Dispatch
- Purpose: one-tap emergency requests and FCFS dispatch.
- Features: guest-allowed request, GPS capture, hospital notifications, accept/assign flow, ETA.
- DB: emergency_requests, hospital_availability.
- API: POST /emergency, POST /emergency/:id/accept, PATCH /emergency/:id/status.
- Edge: spam prevention (rate limits), FCFS concurrency solved with DB locking.
- Permissions: public create, hospital accept.

## 3.9 Notifications & Audit Logs
- Purpose: real-time notifications plus immutable audit logs for compliance.
- Features: in-app/push/email/SMS notifications; audit events stored append-only.
- DB: notifications, audit_logs.
- API: GET /notifications, POST /notifications/mark_read, GET /audit-logs (admin).
- Security: audit logs immutable and restricted.

---

# 4. Non-Functional Requirements
- Performance: <200ms median API latency for common endpoints; appointment search <100ms.
- Security: TLS everywhere, password hashing (argon2/bcrypt), refresh token rotation, secrets in secret manager.
- Scalability: stateless API servers; Redis for cache & queues; PostgreSQL primary + read replicas.
- Reliability: 99.9% uptime target for core services; retry & idempotency on external calls.
- Maintainability: modular code, tests, documented API (OpenAPI).
- Accessibility: WCAG 2.1 AA for patient-facing UI.
- Responsiveness: mobile-first design & animations tuned.
- SEO: public marketing pages SEO-friendly (server-side rendered or pre-rendered).
- Cross-browser: modern browsers + evergreen mobile browsers.
- Logging: structured logs (JSON) with request ID, user ID, error trace.
- Monitoring: Sentry, Prometheus, Grafana, alerting.
- Backup: daily DB backups + WAL archiving; periodic restore drills.
- Recovery: defined RTO & RPO (e.g., RTO <1 hour, RPO < 1 hour).

---

# 5. Complete User Flow (Detailed)
Landing → Registration → Email OTP Verification → Login → Onboarding (select hospital/tenant) → Dashboard → Book Appointment → Receive Confirmation → Check-in / Queue → Consultation → Prescription & Ordered Labs → Lab Uploads Report → Billing Generated → Payment → Receipt → Follow-ups & Reminders → Profile → Logout.

Branch flows:
- Forgot Password → Reset → Login.
- Emergency (guest) → Hospital Accept → Ambulance Dispatch.
- Payment Fail → Retry Flow → Dispute.
- Reschedule Flow → Notifications to both parties.
- Admin change lab price → Approval Workflow → Audit Log.

---

# 6. Role Based Flow (Detailed)
For each role list permissions, restrictions, dashboard items, and API usage.

## Super Admin
- Permissions: full cross-tenant access, manage tenants & platform configs.
- Dashboard: metrics, tenant list, incidents, audit logs.
- API: tenant management endpoints.

## Admin (Hospital)
- Permissions: manage users within tenant, manage lab pricing, view billing & audit logs, approve session limits.
- Dashboard: finance dashboard, lab price approval, user management.
- API: user CRUD, lab-pricing endpoints.

## Doctor
- Permissions: view schedule, manage appointments (status), create prescriptions, order lab tests.
- Dashboard: today's appointments, patient queue, e-prescribe.
- API: appointments read/write, prescriptions create.

## Lab Staff
- Permissions: view assigned lab orders, upload reports, change status.
- Dashboard: pending tests, upload UI, lab pricing view.
- API: lab-tests endpoints.

## Pharmacist
- Permissions: view prescriptions, create pharmacy orders, mark pickup/delivery.
- Dashboard: pending pharmacy orders.
- API: pharmacy-orders endpoints.

## Patient
- Permissions: book appointments, view own data (prescriptions, reports, bills), pay bills.
- Dashboard: upcoming appointments, payment history, prescriptions.
- API: limited user-scoped endpoints.

## Ambulance Partner
- Permissions: view & accept emergency requests for assigned hospitals.
- Dashboard: accept/ETA management.
- API: emergency endpoints.

---

# 7. Information Architecture
- Sitemap:
  - / (marketing)
  - /login, /register, /verify, /forgot
  - /app/dashboard
    - /app/appointments
    - /app/doctors
    - /app/labs
    - /app/prescriptions
    - /app/billing
    - /app/emergency
    - /app/notifications
    - /app/users (admin)
    - /app/settings
- Navigation: top bar (search, notifications, profile), side nav for app pages.
- Breadcrumbs: maintain context and deep linking (e.g., /app/appointments/:id).
- Deep linking: each resource has stable uuid in URL; query params for filters.

---

# 8. Complete Screen List (Representative — each with details)
For brevity we list the screens and core behaviors expected.

1. Landing — marketing content, CTA, signup/login.
2. Login — inputs (email, password), validation, forgot link.
3. Register — role selection, OTP flow, password policy enforcement.
4. Email Verify — OTP entry or verification link.
5. Dashboard (role-aware) — widgets, quick actions, KPIs.
6. Doctor Calendar — day/week view, appointment details, status actions.
7. Booking Wizard — search, select doctor, pick slot, confirm, optional payment.
8. Appointment Detail — patient info, status change, notes, queue info.
9. Lab Order — tests listing, price display, place order.
10. Lab Upload — file upload UI, progress, validation messages.
11. Prescription Editor — diagnosis input, medicines nested, schedule builder.
12. Prescription Viewer — formatted prescription, download, send to pharmacy.
13. Billing & Invoice Viewer — line items, pay button, receipt.
14. Emergency Button — big, accessible, 3-second hold behavior, immediate feedback.
15. Emergency Dashboard (hospital) — incoming requests, accept, ETA.
16. Notifications Center — list, filter, mark read.
17. User Management (admin) — list, create, role assign, deactivate.
18. Audit Logs — append-only logs, filters, admin-only.
19. Settings — hospital configs, payment configs, webhooks.
20. Profile — personal info, change password, sessions list.
21. Reports & Analytics — revenue, utilization, AI insights.
22. Onboarding Wizard (tenant) — setup hospital settings, session limits.
23. Error & Maintenance Pages — friendly messaging.

Each screen includes loading, empty, success, error states and responsive behaviors.

---

# 9. Database Design (PostgreSQL) — Key Tables & Indexes
Use UUIDs (v4) for primary keys.

Core tables:
- users (id PK, email unique, password_hash, name, role enum, tenant_id, specialty, is_active, email_verified, created_at, updated_at, deleted_at)
- tenants (id, name, metadata, created_at)
- sessions (id, user_id FK, device, ip, created_at, last_seen)
- refresh_tokens (id, user_id FK, token_hash, expires_at, revoked)
- appointments (id, doctor_id FK, patient_id FK, date, time, queue_number, status, reason, created_at, updated_at)
  - Unique constraint: (doctor_id, date, time)
  - Index: (doctor_id, date)
- appointment_slots (optional)
- prescriptions (id, patient_id FK, doctor_id FK, diagnosis, created_at)
- prescription_items (id, prescription_id FK, medicine_name, dosage, frequency, duration)
- lab_pricing (id, hospital_id FK, test_name, price, approval_required, created_at, updated_at)
- lab_tests (id, patient_id FK, doctor_id FK, test_name, price, status, report_file_key, requested_at, completed_at)
- billing (id, patient_id FK, items jsonb, total_amount, paid_status boolean, verified_badge boolean, created_at)
- payment_orders (id, provider_order_id unique, bill_id FK, amount, currency, status, metadata jsonb)
- payment_transactions (id, order_id FK, provider_payment_id, status, created_at)
- pharmacy_orders (id, prescription_id FK, patient_id, items jsonb, status, created_at)
- emergency_requests (id, patient_id nullable, lat, lng, status, accepted_hospital_id, eta, created_at)
- notifications (id, user_id, message, type, read_status boolean, created_at)
- audit_logs (id, actor_user_id nullable, action_type, resource_type, resource_id, detail jsonb, created_at) — append-only
- session_limits (id, doctor_id FK, limit int)

Indexes & constraints:
- Index users(email), users(tenant_id)
- Partial index billing (paid_status = false)
- Foreign keys with ON DELETE SET NULL for some relations to preserve audit trails.
- Soft delete via deleted_at column on main user/tenant objects.

---

# 10. Prisma Schema Planning
- Use schema.prisma with models defined to mirror DB design.
- Use `@@index`, `@@unique` and field-level `@default` as appropriate.
- Use enums for Role, AppointmentStatus, LabTestStatus, PaymentStatus.
- Migration strategy: use `prisma migrate dev` in development, `prisma migrate deploy` in CI for production.
- Naming conventions: PascalCase model names; snake_case db names if set via map, consistent across codebase.

Example model (abridged):
```prisma
model User {
  id           String   @id @default(uuid())
  email        String   @unique
  passwordHash String
  name         String
  role         Role
  specialty    String?
  tenantId     String
  isActive     Boolean  @default(true)
  emailVerified Boolean @default(false)
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
  deletedAt    DateTime?
  @@index([tenantId])
}

enum Role {
  SUPER_ADMIN
  ADMIN
  DOCTOR
  LAB
  PHARMACIST
  PATIENT
  AMBULANCE
}