# Healthcare+ — Phase Plan: Phases 11 → 18 (Detailed Implementation Guide)

This document contains a detailed breakdown for Phases 11 through 18. Each phase includes: goals, deliverables, required DB/model changes, API surface, frontend work, background jobs/workers, infra/dependency needs, security considerations, testing checklist, estimated complexity & timebox, and a clear Definition of Done (DoD).

---

## Phase 11 — Lab Pricing & Ordering

### Goals
- Implement lab pricing management and ordering flow with admin approval for price changes.
- Provide APIs and UI to list available tests with fixed pricing, and allow doctors to order tests for patients.
- Ensure every pricing change is auditable.

### Deliverables
- DB models: `lab_pricing`, approval/audit records.
- Backend APIs:
  - GET `/api/v1/lab-pricing` (list public pricing for tenant/hospital)
  - POST `/api/v1/lab-pricing` (admin create)
  - PATCH `/api/v1/lab-pricing/:id` (admin update → triggers approval flow)
  - POST `/api/v1/lab-tests` (doctor places order)
  - GET `/api/v1/lab-tests?patient_id&status`
- Frontend:
  - Admin Lab Pricing page (list, create, edit, propose changes)
  - Doctor Lab Ordering flow integrated into appointment/prescription pages
  - Patient view: list of ordered tests + pricing shown on patient dashboard
- Approval workflow: price changes go into `price_change_requests` with status [PENDING, APPROVED, REJECTED]
- Audit logging: each change recorded in `audit_logs`.

### DB Impact (schema changes)
- `lab_pricing`:
  - id (uuid PK), hospital_id, test_code (string, unique per hospital), test_name, price (numeric), currency, active (bool), created_at, updated_at
  - `approval_required` boolean (default true)
- `price_change_requests`:
  - id, lab_pricing_id, requested_by (user_id), old_price, new_price, status, reason, created_at, resolved_by, resolved_at
- `lab_tests` (existing): ensure fields include `pricing_snapshot` (jsonb) or `price_at_order` numeric to freeze price at order time.

### API Requirements
- Input validation: test_code, price > 0, currency allowed values.
- Authorization:
  - Pricing creation/update: Admin role only.
  - Ordering lab: Doctor role (and optionally nurse) only.
  - Patients can view their own lab orders.
- Response format: consistent envelope; include `pricing_snapshot` in lab order response.

### Frontend Screens
- Admin: Lab Pricing List, Create Pricing Modal, Request Price Change Modal, Price Change Request List with Approve/Reject actions.
- Doctor: Order Lab modal integrated in patient chart; displays pricing and totals.
- Patient: Ordered Tests list + status.

### Background Jobs
- Notification job: notify admins of pending price change requests.
- Optionally: scheduled sync job to reconcile pricing exports to accounting.

### Security & Compliance
- Ensure only admins can propose/approve pricing.
- All pricing changes recorded in `audit_logs` with actor, timestamp, and reason.
- Validate and sanitize all input to avoid injection.

### Testing Checklist
- Unit tests for pricing CRUD and request lifecycle.
- Integration test: Doctor orders a lab → `lab_tests` created with frozen price.
- Authorization tests: non-admin cannot update pricing.
- Edge-case tests: price set to 0 or negative rejected.

### Estimated Complexity & Timebox
- Complexity: Medium
- Dev effort: 1 backend dev + 1 frontend dev for ~1–2 sprints (2–4 weeks).

### Definition of Done
- APIs implemented and documented (OpenAPI).
- Admin and Doctor UIs implemented with validation.
- Price change requests require explicit approve/reject and generate audit log.
- Tests (unit + integration) added and passing.

---

## Phase 12 — Lab Test Processing & Secure Upload

### Goals
- Implement secure, robust lab report upload and processing pipeline.
- Use presigned uploads to S3-compatible storage; enforce type/size/scan policies.
- Ensure uploaded reports are linked to lab tests, processed asynchronously, and notifications sent.

### Deliverables
- Backend:
  - Endpoint to create presigned upload URL: POST `/api/v1/lab-tests/:id/presign-upload`
  - Endpoint to mark upload complete / process (webhook or worker-triggered)
  - Worker to process uploaded file: verify PDF, OCR/metadata extraction (optional), virus scan, generate thumbnails/preview
  - Endpoint to download / generate presigned download links: GET `/api/v1/lab-tests/:id/report-url`
- Frontend:
  - Upload UI with progress, client-side file validations (size, type), preview for PDFs.
  - Lab staff dashboard to see pending uploads and process results.
  - Patient report viewer (embedded PDF viewer or modal).
- Storage:
  - S3 bucket with restricted policies; use signed URLs for upload & download.
- Background jobs:
  - File scan (ClamAV or cloud-managed antivirus)
  - Post-processing (OCR/layout extraction) as asynchronous jobs
  - Notification sending when processing completes

### DB Impact
- `lab_files` table:
  - id, lab_test_id FK, storage_key, file_name, content_type, size, uploaded_by, uploaded_at, processed_at, processing_status, metadata jsonb
- `lab_tests.report_file_key` should reference `lab_files.id` or contain presigned key.

### API Requirements & Validations
- Presign: check caller authorization (lab staff or service); ensure size <= 50MB (configurable).
- Upload complete: verify presence of file in storage (optionally via S3 HEAD) and create `lab_files` record.
- Download presign: verify user ownership or role permission before returning presigned GET.

### Security & Data Protection
- All S3 objects encrypted server-side (SSE).
- Sensitive files access controlled via short-lived presigned URLs (e.g., 5–15 minutes).
- Virus scanning mandatory before making file accessible to patients.
- Filename sanitization; store files under UUID keys, never store original filename as public key.

### Background & Worker Considerations
- Use BullMQ (Redis) with separate queues: `file-scans`, `lab-processing`, `thumbnail-generation`.
- Retry policy for transient failures; dead-letter queue for manual review.

### Testing Checklist
- Test presign flow: presign generated only for authorized users and correct size/content.
- Simulate upload and worker processing; ensure processed status and notifications.
- Security tests for unauthorized download attempt.
- Large file handling & retry tests.

### Estimated Complexity & Timebox
- Complexity: High (due to external storage & scanning)
- Dev effort: 2 backend devs + 1 frontend dev, ~3–6 weeks.

### Definition of Done
- Presigned upload + download implemented and secure.
- Files scanned and processed by worker; processing status stored and surfaced in UI.
- Patient can access report only after successful processing and with correct permissions.
- Tests and monitoring for worker failures in place.

---

## Phase 13 — Billing Model & Invoice Generation

### Goals
- Implement itemized billing engine that creates invoices from services (consultation, labs, pharmacy) with transparent line items and immutable records.
- Provide endpoints and UI for bill creation, retrieval, and administrative adjustments (with audit).

### Deliverables
- Billing API:
  - POST `/api/v1/billing` — create bill (admin/system)
  - GET `/api/v1/billing/:id` — get invoice
  - GET `/api/v1/billing?patient_id=...` — patient bills
  - PATCH `/api/v1/billing/:id` — limited edits (audit required)
- Invoice generation:
  - Generate PDF receipt (server-side) and store/report in `billing.invoice_file_key`
- DB:
  - `billing` table: id, patient_id, items jsonb, subtotal, taxes, discounts, total_amount, currency, status (DRAFT, ISSUED, PAID, REFUNDED), created_by, created_at, invoice_file_key
  - `billing_line_items` (optional for normalized queries)
- Frontend:
  - Invoice viewer (line items, totals)
  - Admin create bill UI & adjustments
  - Patient pay UI (links to payment flow)
- Business logic:
  - Link bill to appointment/lab_order/pharmacy_order with references for traceability
  - Price locking: prices used must be snapshot at order creation
- Audit:
  - Any edit to issued invoices creates `billing_adjustments` record and `audit_logs` entry

### Validation & Rules
- Validate items have valid price and quantity; totals computed server-side (not trusted from client).
- Disallow editing after status `PAID` except via credit/refund flows with admin approval & audit.

### Background Jobs
- Periodic job to reconcile payment statuses (for external providers).
- Batch job to send overdue payment reminders.

### Security & Compliance
- Bills contain PHI elements; access control strict — only patient, billing admins, and authorized staff can view.
- Immutable audit trail of all modifications.

### Testing Checklist
- Create bill from combined sources (appointment + lab + pharmacy) and verify totals.
- PDF invoice generation correctness test (line items present).
- Edit bill attempts post-paid should be blocked or create adjustment records.
- Integration tests linking billing → payments (Phase 14).

### Estimated Complexity & Timebox
- Complexity: Medium–High (PDF generation + linking)
- Dev effort: 2 backend devs + 1 frontend dev, ~2–4 weeks.

### Definition of Done
- Bills can be created, viewed, and downloaded as PDF.
- Bills link to source orders; edits produce audit records.
- Tests for bill total calculation and status transitions exist.

---

## Phase 14 — Payment Integration & Webhooks

### Goals
- Integrate a production-grade payment provider (Stripe recommended globally; Razorpay option for India).
- Ensure secure order creation, webhook verification, idempotency, and persistence of payment events.

### Deliverables
- Payment models:
  - `payment_orders` (id, provider_order_id, bill_id, amount, currency, status, metadata, created_at)
  - `payment_transactions` (id, order_id, provider_payment_id, status, received_at, raw_payload)
- Backend endpoints:
  - POST `/api/v1/payments/create-order` — create order record and return provider client token/order details
  - POST `/api/v1/payments/verify` — optional endpoint used by frontend for immediate verification flows (but main verification via webhook)
  - POST `/api/v1/payments/webhook` — provider webhook endpoint (must verify signature)
- Frontend:
  - Payment flow component integrating provider SDK (Stripe Elements or Razorpay Checkout)
  - UX for success/failure; spinner and idempotency handling
- Webhook handling:
  - Validate signature of provider webhook using keys from secret manager
  - Find `payment_orders` by provider_order_id or metadata; update statuses atomically
  - Ensure idempotency (ignore duplicate webhook deliveries)
  - Log raw webhook payload to `payment_transactions.raw_payload` for audit

### Security & Best Practices
- Store provider keys in secret manager; do not hardcode or fallback to mocks in production.
- Always verify webhook signatures; never bypass verification for staging (provide staging keys and test-mode flows).
- Use idempotency keys on outbound requests to providers to prevent duplicate transactions.
- Persist orders BEFORE redirecting user to provider checkout.

### Error Handling & Reconciliation
- On webhook errors, push to dead-letter queue for manual reconciliation.
- Implement daily reconciliation job between `payment_orders` and provider reports.

### Testing Checklist
- Test create-order flow and provider token response handling.
- Simulate webhooks with both valid and invalid signatures verifying behavior.
- Idempotency test: duplicate webhook delivery should not double-mark paid.
- Refund flow test (if supporting refunds) and partial refund behavior.

### Estimated Complexity & Timebox
- Complexity: High
- Dev effort: 2 backend devs + 1 frontend dev + QA, ~3–5 weeks.

### Definition of Done
- Payment creation, client-side checkout and webhook verification fully implemented.
- Orders persisted and linked to bills; idempotency validated.
- Reconciliation and monitoring alarms in place for webhook failures.

---

## Phase 15 — Pharmacy Orders & Fulfillment Flow

### Goals
- Implement pharmacy order lifecycle triggered from prescriptions and driven by pharmacists for fulfillment.
- Support pickup and delivery statuses and receipt generation.

### Deliverables
- DB:
  - `pharmacy_orders` (id, prescription_id, patient_id, items jsonb, total_amount, status [PENDING, PROCESSING, READY, PICKED_UP, DELIVERED, CANCELLED], created_at)
  - `pharmacy_items` (optional normalized)
- Backend APIs:
  - POST `/api/v1/pharmacy-orders` — create from prescription (patient or pharmacist)
  - GET `/api/v1/pharmacy-orders?patient_id&status`
  - PATCH `/api/v1/pharmacy-orders/:id/pickup` — mark picked-up
  - PATCH `/api/v1/pharmacy-orders/:id/deliver` — mark delivered
- Frontend:
  - Patient-facing: confirm pharmacy order from prescription, payment link if prepay required
  - Pharmacist-facing: order queue, accept/process UI, pick/pack flow, mark pickup/delivery
- Notifications:
  - Notify patient when order is ready and when picked up/delivered
- Integrations:
  - Optional delivery partner integrations (webhook or API)
  - Inventory management (future: check stock before confirming)

### Business Logic & Validation
- Price check: line item prices verified against `pharmacy_prices` table (or vendor).
- Partial fulfillment: support partial availability with backorder flags.
- Refunds: if not fulfilled or cancelled — create billing credit or refund via payment provider.

### Background Jobs
- Send reminders for pickup after N hours/days
- Auto-cancel unpaid orders after TTL

### Testing Checklist
- End-to-end flow from prescription → pharmacy order → pickup.
- Partial fulfillment scenarios.
- Notification delivery & status transitions.

### Estimated Complexity & Timebox
- Complexity: Medium
- Dev effort: 1–2 backend devs + 1 frontend dev, ~2–3 weeks.

### Definition of Done
- Pharmacy orders can be created and processed with status transitions.
- Patient and pharmacist UIs implemented; notifications work.
- Edge cases (cancellations, partial fills) supported and audited.

---

## Phase 16 — Emergency Flow & FCFS Enforcement

### Goals
- Implement reliable emergency request/dispatch flow that supports guest-initiated requests and First-Come-First-Serve (FCFS) assignment to hospitals.
- Ensure concurrency-safe acceptance to avoid multiple hospitals being assigned to the same request.

### Deliverables
- DB:
  - `emergency_requests` (id, patient_id nullable, lat, lng, address, status [REQUESTED, ACCEPTED, ASSIGNED, CANCELLED, COMPLETED], accepted_hospital_id nullable, accepted_at, eta, created_at)
  - `hospital_availability` or `hospital_profiles` reflecting registered hospitals and availability
- Backend APIs:
  - POST `/api/v1/emergency` — create emergency request (public endpoint, rate-limited)
  - GET `/api/v1/emergency/:id` — status
  - POST `/api/v1/emergency/:id/accept` — hospital accepts (must be authenticated hospital staff)
  - PATCH `/api/v1/emergency/:id/status` — updates (ETA, enroute, completed)
- Frontend:
  - Patient: emergency button (3-second hold), confirmation modal, live status updates
  - Hospital: emergency queue view, accept button (with time & ETA input)
  - Ambulance partner UI: received assignment, ETA updates
- Concurrency & FCFS Enforcement:
  - Acceptance must be atomic: use DB transactional locking (SELECT FOR UPDATE) or optimistic locking and verify acceptance timestamp is earliest.
  - Use single source-of-truth (DB) to write `accepted_hospital_id` only once
- Notifications:
  - Notify nearest hospitals on request creation (via push/webhook)
  - Notify requester when accepted & ETA set
- Abuse Mitigation:
  - Rate-limit public emergency endpoint (e.g., 1 per N minutes per IP/device)
  - Add simple CAPTCHA/behavior detection for repeated abuse
- Security & Privacy:
  - Public endpoint should only accept minimal data; any PII stored must be protected
  - Ensure logs do not expose excessive location history beyond retention policy

### Background Jobs
- Dispatch outreach job: notify set of hospitals asynchronously
- TTL job to auto-cancel stale requests or reassign if acceptance expired

### Testing Checklist
- Concurrency tests: simulate multiple accepts; ensure exactly one hospital wins FCFS
- Rate-limiting tests for public endpoint
- Acceptance & ETA updates reflected correctly
- Notification tests for hospitals and requester

### Estimated Complexity & Timebox
- Complexity: High (concurrency, real-time notifications)
- Dev effort: 2 backend devs + 1 frontend dev + infra (push) support; ~3–5 weeks.

### Definition of Done
- Emergency request can be created by guest; hospitals notified.
- FCFS enforced reliably under concurrent accept attempts.
- Requester receives acceptance notification with ETA.
- Abuse controls & request TTL handling present.
- Tests for concurrency and rate-limiting pass.

---

## Phase 17 — Audit Logging & Compliance Controls

### Goals
- Implement a robust, append-only audit logging mechanism covering all sensitive actions (auth events, billing changes, price changes, payment events, upload/download of PHI, role changes).
- Provide admin UI to search/filter logs and export for compliance/audit reviews.
- Implement data retention & masking policies suitable for regulatory requirements.

### Deliverables
- DB & Model:
  - `audit_logs` (id, tenant_id, actor_user_id, action_type [enum], resource_type, resource_id, detail jsonb, severity, created_at)
  - Ensure entries are append-only; prevent accidental deletion via DB permissions or application-level guard rails
- Backend:
  - Audit helper functions to write logs in a consistent format
  - Middleware to automatically log auth events (login, logout, failed login), permission changes, billing changes, payment events, file uploads/downloads of lab reports
  - Admin API: GET `/api/v1/audit-logs` with filters (action_type, date range, resource_type, actor)
- Frontend:
  - Admin Audit Logs page with advanced filters, search, and export (CSV/PDF) capabilities
- Compliance:
  - Data retention policy configuration (e.g., retain logs for minimum X years)
  - PII masking rules when exporting (option to mask patient identifiers)
  - Role-based access to logs — only admins & compliance officer roles allowed
- Security:
  - Ensure audit_logs writes are immutable from the app: do not provide API to delete logs
  - DB-level recommendations: restrict delete permissions; consider writing logs to an append-only store or S3 with immutability if required by regulation

### Testing Checklist
- Write & retrieve audit logs for each critical action in system tests
- Verify logs contain full context necessary for an audit (who/what/when/where/why)
- Export test: ensure mask options work and exported data matches filters

### Estimated Complexity & Timebox
- Complexity: Medium
- Dev effort: 1 backend dev + 1 frontend dev, ~2–3 weeks.

### Definition of Done
- All critical actions generate audit entries.
- Admin UI available with filtering and export.
- Log retention & masking policy implemented.
- Tests validate immutability and log completeness.

---

## Phase 18 — Observability & Monitoring

### Goals
- Implement full observability stack: metrics, tracing, error tracking, and alerting to ensure operational health and rapid incident response.
- Provide dashboards and runbooks for common incidents.

### Deliverables
- Instrumentation:
  - Request/response metrics: latency, error rate, throughput per endpoint
  - Business metrics: booking rate, payment success rate, queue backlog
  - Worker metrics: queue size, job failures, retries
- Error tracking:
  - Sentry integration capturing exceptions, stack traces, and user context (no raw PHI in logs)
- Metrics & Dashboard:
  - Prometheus metrics exposed via `/metrics`
  - Grafana dashboards: API health, DB performance, worker health, payments, emergency requests
- Tracing:
  - Distributed tracing using OpenTelemetry (optional) to trace cross-service requests (frontend → backend → worker → external providers)
- Logging:
  - Structured logs (pino/winston) with correlation ID (X-Request-ID)
  - Centralized log storage (ELK/Cloud logging) with retention policy
- Alerts:
  - Alert rules for error rate threshold, high latency, worker queue backlog > threshold, failed webhook deliveries, high payment failure rate
  - Integrations: PagerDuty/Slack/Email for on-call notifications
- Runbooks:
  - Incident runbooks for common scenarios: payment webhook failure, worker backlog, DB connection issues, emergency flow failure
- Health checks:
  - Readiness & liveness endpoints for container orchestration
  - Health checks for DB connectivity, Redis, S3 access, provider connectivity

### Implementation Steps
1. Add request middleware to start/propagate correlation ID.
2. Integrate pino with JSON output and send logs to central collector.
3. Export Prometheus metrics using a lightweight metrics library.
4. Integrate Sentry SDK and capture exceptions in production/staging.
5. Configure Grafana dashboards with key panels and set alert thresholds.
6. Add health check endpoints for orchestrator probes.

### Security & Privacy Considerations
- Do NOT log sensitive fields or raw PHI; mask or hash identifiers in logs.
- Ensure telemetry data stored in a secure environment.
- Access to dashboards and logs should be RBAC-protected.

### Testing Checklist
- Tests for metric increment when endpoints invoked.
- Synthetic transactions to ensure alerts fire correctly (simulate increased error rate).
- Validate Sentry captures sample exceptions and attaches context.

### Estimated Complexity & Timebox
- Complexity: Medium–High (depending on breadth)
- Dev effort: 1 backend dev + SRE/DevOps support, ~2–4 weeks.

### Definition of Done
- Prometheus metrics, Grafana dashboards, and Sentry integrated and configured.
- Alerting channels configured and tested.
- Runbooks available and documented for primary incidents.
- Health checks implemented and used by orchestrator.

---

## Final Notes for Phases 11–18
- Each phase assumes Phases 1–10 are already implemented and stable (auth, core appointment flows, basic UI).
- For each phase:
  - Create a scoped set of JIRA tickets (or your task tracker) grouped by backend, frontend, infra, QA.
  - Create feature branches with required feature flags for safe rollout.
  - Add acceptance tests and include them in CI pipeline.
  - Deploy to staging first and run smoke & integration tests before rolling into production.
- Security & compliance should be reviewed at the end of each phase, especially when handling PHI, billing, or payments.

---

If you want, I can:
- Scaffold the backend APIs and Prisma migrations for any one of these phases (pick a phase).
- Create a checklist/prioritized ticket list per phase formatted for your issue tracker (JIRA/GitHub Issues).
- Draft the OpenAPI (swagger) spec for the new endpoints in the selected phase.

Which phase should I scaffold first (11–18)?