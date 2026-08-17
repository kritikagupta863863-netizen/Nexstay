# NEXSTAY PG Management ERP — Software Requirements Specification

Status: Draft v1
Scope: owner-side ERP only (see [PRD.md](PRD.md) §1)
Implementation plan: [BUILD_GUIDE.md](BUILD_GUIDE.md)
Conventions: SHALL = mandatory. Requirement IDs `SRS-<area>-<n>` are referenced by tests.

---

## 1. Introduction

### 1.1 Purpose
This document specifies the software behaviour of the NEXSTAY PG Management ERP precisely enough to implement and test it without further product decisions. Product intent and prioritisation live in the PRD; this document defines interfaces, data, algorithms, state machines and quality attributes.

### 1.2 Definitions
| Term | Meaning |
| --- | --- |
| Org | Organization — tenant boundary of the system (an owner's business) |
| Property | One PG building belonging to an Org |
| Bed | Smallest billable physical unit |
| Tenancy | Contract linking a Tenant to a Bed for a period |
| Billing period | The month (or cycle) an invoice covers |
| Billing day | Day of month invoices are generated for a tenancy |
| Paise | 1/100 rupee; the storage unit for all monetary values |
| Ledger | Append-only record of financial events |
| BSP | WhatsApp Business Solution Provider |

### 1.3 System context
```
        ┌──────────────────────────┐
Owner → │  Web app (Next.js SSR)   │
Staff → │  mobile-responsive       │
        └────────────┬─────────────┘
                     │ HTTPS / JSON (REST)
        ┌────────────▼─────────────┐        ┌──────────────────┐
        │   API (NestJS modular    │───────►│ Postgres         │
        │   monolith)              │        └──────────────────┘
        │  auth · properties ·     │        ┌──────────────────┐
        │  tenancies · billing ·   │───────►│ Redis + BullMQ   │
        │  expenses · complaints · │        └──────────────────┘
        │  employees · notices ·   │        ┌──────────────────┐
        │  analytics · subscription│───────►│ S3-compatible    │
        └───┬────────────┬─────────┘        │ object storage   │
            │            │                  └──────────────────┘
            ▼            ▼
   WhatsApp/SMS BSP   PDF renderer
```
Tenants are **not** users of this system; they are notified through the BSP and may open read-only tokenised links.

---

## 2. Overall description

### 2.1 Product perspective
A multi-tenant SaaS web application. Org is the isolation boundary; every domain row carries `org_id` and every query is scoped by it.

### 2.2 User classes
Roles and permissions per PRD §2, enforced server-side (see §7.2).

### 2.3 Operating environment
- Clients: Chrome/Safari on Android and iOS phones (primary), desktop Chrome/Edge/Safari (secondary). Last two major versions.
- Server: Linux containers; PostgreSQL 15+; Redis 7+; Node 20 LTS.
- Region: India (data residency, `Asia/Kolkata` for all business-day logic).

### 2.4 Design constraints
- **C-1** All monetary values are integers in paise; no floating point arithmetic on money anywhere.
- **C-2** No cross-org data access at the data layer; enforced by a query-level scope plus Postgres row-level security as defence in depth.
- **C-3** Issued invoices are immutable.
- **C-4** All background jobs are idempotent and safe to re-run.
- **C-5** All timestamps stored as UTC `timestamptz`; all date-based business rules evaluated in `Asia/Kolkata`.
- **C-6** Every list endpoint is paginated (cursor-based) and filtered server-side.

### 2.5 Assumptions and dependencies
External dependencies: BSP for WhatsApp/SMS, object storage, PDF rendering, (later) payment gateway. Each SHALL sit behind an interface with a no-op/stub implementation so the system is testable and deployable without it.

---

## 3. Architecture

### 3.1 Modules (bounded contexts)
| Module | Responsibility | Depends on |
| --- | --- | --- |
| `iam` | Org, users, roles, OTP auth, sessions, audit log | — |
| `properties` | Property, floor, room, bed, occupancy projection | iam |
| `tenants` | Tenant records, documents, ID proof encryption | iam |
| `tenancies` | Tenancy lifecycle, rent revisions, transfers, notice, move-out settlement | properties, tenants |
| `billing` | Invoices, line items, numbering, cycle runs, late fees, credit notes | tenancies, utilities |
| `payments` | Payment recording, allocation, receipts, dues, `PaymentProvider` interface | billing |
| `utilities` | Meters, readings, electricity computation | properties, tenancies |
| `expenses` | Expenses, categories, recurring templates | properties |
| `complaints` | Complaints, events, assignment, SLA | properties, tenancies, employees |
| `employees` | Employees, attendance, salary, salary→expense posting | properties, expenses |
| `notices` | Notice composition, audience resolution, delivery log, reminder schedules | tenancies, messaging |
| `messaging` | Channel abstraction (WhatsApp/SMS/none), templates, retries | — |
| `documents` | Uploads, signed URLs, PDF generation | — |
| `analytics` | Dashboard aggregates, reports, exports | billing, payments, expenses, properties |
| `subscription` | Plans, bed metering, limits, dunning | iam, properties |

Rule: modules communicate through service interfaces and domain events, never by reaching into each other's tables. This is what makes the deferred marketplace/tenant-app work additive.

### 3.2 Domain events (in-process, persisted to an outbox)
`TenancyStarted`, `TenancyEnded`, `RentRevised`, `BedTransferred`, `InvoiceIssued`, `InvoiceCancelled`, `PaymentRecorded`, `ComplaintCreated`, `ComplaintStatusChanged`, `ReadingRecorded`, `NoticeSent`, `BedCountChanged`.
Consumers: notices (reminders/notifications), analytics (cache invalidation), subscription (bed metering), audit.

### 3.3 Technology decisions
| Concern | Choice | Rationale |
| --- | --- | --- |
| API | NestJS + TypeScript | Module boundaries and DI enforce §3.1; shares types with the web app |
| Web | Next.js (App Router) + Tailwind + shadcn/ui | SSR for fast phone loads; one language across the stack |
| DB | PostgreSQL | Relational integrity for money and occupancy; window functions for reports |
| ORM/migrations | Prisma (or TypeORM) with checked-in SQL migrations | Reviewable schema history |
| Jobs | BullMQ on Redis | Cron-style invoice runs and reminder fan-out with retries |
| Files | S3-compatible (Cloudflare R2/S3) | Signed URLs for private documents |
| PDF | Headless Chromium (Puppeteer) rendering an HTML template | Invoice layout is HTML/CSS, easy to iterate |
| Messaging | BSP (MSG91/Gupshup) behind `MessagingProvider` | Templates need provider approval; SMS fallback |
| Auth | Phone OTP, JWT access + refresh, httpOnly cookies | Owners have no patience for passwords |
| Observability | Sentry + structured JSON logs + job dashboards | Failed invoice runs must alert loudly |

---

## 4. Functional requirements

### 4.1 Authentication and identity (`iam`)
- **SRS-AUTH-1** The system SHALL authenticate users by phone number + 6-digit OTP valid for 5 minutes, single-use.
- **SRS-AUTH-2** OTP requests SHALL be rate-limited to 3 per phone per 10 minutes and 10 per phone per day; 5 consecutive wrong OTPs SHALL lock OTP login for that phone for 30 minutes.
- **SRS-AUTH-3** On success the system SHALL issue an access token (15 min) and a refresh token (30 days, rotating, revocable per device).
- **SRS-AUTH-4** A phone number MAY belong to users in multiple orgs; if more than one membership exists, the client SHALL prompt for org selection and the token SHALL be scoped to the selected org.
- **SRS-AUTH-5** Owners MAY additionally set an email + password (argon2id) as a fallback login.
- **SRS-AUTH-6** Users SHALL be invitable by phone with a role; invitations expire in 7 days.
- **SRS-AUTH-7** Deactivating a user SHALL revoke all their sessions within 60 seconds.
- **SRS-AUTH-8** Internal admin impersonation SHALL require a reason, be time-boxed to 60 minutes, and be audit-logged and visible to the owner.

### 4.2 Properties, rooms, beds (`properties`)
- **SRS-PROP-1** An Org SHALL support N properties; every operational entity SHALL reference exactly one property.
- **SRS-PROP-2** Room creation SHALL support bulk creation from a numeric range plus a shared template (sharing type, rent, tags), creating `sharing_capacity` beds per room labelled `<room>-A…`.
- **SRS-PROP-3** Increasing a room's `sharing_capacity` SHALL append beds; decreasing it SHALL be rejected with `409 ROOM_CAPACITY_BELOW_OCCUPANCY` listing blocking tenancy IDs, unless the beds to be removed are vacant.
- **SRS-PROP-4** Bed status SHALL be derived, not stored authoritatively, by:
  ```
  status(bed, on_date) =
    Blocked        if an active block covers on_date
    Occupied       if ∃ tenancy where move_in ≤ on_date and (move_out is null or move_out ≥ on_date) and notice_date is null
    NoticePeriod   if the same but notice_date is not null
    Vacant         otherwise
  ```
  A materialised `bed_occupancy_current` projection MAY be maintained for query speed and SHALL be rebuildable from tenancies by a single command.
- **SRS-PROP-5** Occupancy percentage for a period SHALL be `occupied_bed_days / available_bed_days`, where blocked beds are excluded from available.
- **SRS-PROP-6** Rooms and beds referenced by any invoice SHALL be archivable but not deletable (`409 ENTITY_IN_USE`).

### 4.3 Tenants (`tenants`)
- **SRS-TEN-1** Phone number SHALL be unique per org for tenants; it is the tenant's identity for messaging.
- **SRS-TEN-2** Mandatory fields: name, phone, emergency contact name + relation + phone. All others optional.
- **SRS-TEN-3** ID proof documents SHALL be stored in private object storage; the ID **number** SHALL be encrypted at the field level (AES-256-GCM, key from KMS/env, per-org key derivation) and returned masked (`XXXX XXXX 1234`) to every role except Owner.
- **SRS-TEN-4** Any full ID-number read or document download SHALL create an audit entry.
- **SRS-TEN-5** Tenant import SHALL accept CSV with a header row, validate row-by-row, return a per-row error report, and import atomically per row (valid rows import, invalid rows are reported).
- **SRS-TEN-6** Tenants SHALL be soft-deletable only when they have no active tenancy and no unpaid invoice.

### 4.4 Tenancies (`tenancies`)
- **SRS-TCY-1** A tenancy SHALL carry: tenant, bed, `move_in_date`, `agreed_rent_paise`, `deposit_agreed_paise`, `deposit_received_paise`, `billing_day` (1–28 or `ANNIVERSARY`), `notice_period_days`, `food_included`, `recurring_charges[]`, `status`.
- **SRS-TCY-2** Status machine:
  ```
  ACTIVE ──record notice──► NOTICE_PERIOD ──settle──► CLOSED
     │                            │
     └───────settle───────────────┘         (CANCELLED only if no invoice was ever issued)
  ```
- **SRS-TCY-3** Creating a tenancy on a bed with an overlapping active tenancy SHALL fail `409 BED_ALREADY_OCCUPIED`. Overlap is checked on date ranges, so a future-dated tenancy after a known move-out is allowed.
- **SRS-TCY-4** A tenant SHALL NOT hold two overlapping active tenancies in the same property (`409 TENANT_ALREADY_ACTIVE`).
- **SRS-TCY-5** Rent revision SHALL create a `rent_revision` row (`amount`, `effective_from`, `reason`, `actor`); invoice generation SHALL use the revision in force for each day of the billing period, pro-rating across a mid-period change.
- **SRS-TCY-6** Bed transfer SHALL end occupancy of the source bed and begin occupancy of the target bed on `transfer_date` within the same tenancy, preserving deposit and history; the covering invoice SHALL show both beds with pro-rated amounts.
- **SRS-TCY-7** Recording notice SHALL set `notice_date` and compute `expected_vacate_date = notice_date + notice_period_days`; the actual vacate date MAY differ and SHALL be captured at settlement.
- **SRS-TCY-8** Move-out settlement SHALL compute, in this order, and persist as an immutable settlement record with a PDF:
  ```
  final_rent      = prorate(rent_in_force, days_occupied_in_final_period)
  final_utilities = electricity_for(final reading) + flat/food charges prorated
  outstanding     = Σ unpaid balances of all invoices for the tenancy
  deductions      = Σ owner-entered deductions (damage, cleaning, notice shortfall)
  gross_payable_by_tenant = final_rent + final_utilities + outstanding + deductions
  net = deposit_received − gross_payable_by_tenant
  net > 0 → refund_due(net);  net < 0 → amount_receivable(−net);  net = 0 → settled
  ```
- **SRS-TCY-9** Settlement SHALL be reversible only by an Owner-performed reversal that is audit-logged and creates a compensating record; it SHALL NOT be edited in place.
- **SRS-TCY-10** Closing a tenancy SHALL free the bed effective on the actual vacate date, not on the settlement timestamp.

### 4.5 Billing (`billing`)
- **SRS-BILL-1** A `cycle_run` for (property, period) SHALL create at most one invoice per (tenancy, period). The uniqueness SHALL be enforced by a database unique constraint `(tenancy_id, period_start)` on non-cancelled invoices, not only in application code.
- **SRS-BILL-2** Invoice generation for a tenancy and period SHALL compose line items:
  | Type | Source | Rule |
  | --- | --- | --- |
  | `RENT` | rent revisions in force | pro-rated per day occupied in the period |
  | `ELECTRICITY` | `utilities` module | see SRS-UTIL-3; omitted with a warning if the reading is missing |
  | `FOOD` | tenancy `food_included` / property food mode | flat or omitted |
  | `RECURRING` | tenancy `recurring_charges[]` | flat, pro-rated if the tenancy started mid-period |
  | `ONE_OFF` | owner-entered | as entered |
  | `LATE_FEE` | late-fee rule | see SRS-BILL-8 |
  | `DISCOUNT` | owner-entered | negative amount, reason required |
  | `CARRY_FORWARD` | prior unpaid balance | informational line, SHALL NOT double-count in `amount_due` (see SRS-BILL-4) |
- **SRS-BILL-3** Pro-ration SHALL be `round_to_rupee(amount × days_applicable / days_in_period)` using calendar days, and the invoice SHALL print the day count used.
- **SRS-BILL-4** `invoice.total = Σ line_items(excluding CARRY_FORWARD)`; `invoice.balance_due = total − Σ allocated_payments`. Tenancy-level outstanding is computed across invoices; a carry-forward line is display-only.
- **SRS-BILL-5** Invoice numbering SHALL be per property, gapless, formatted `<prefix>/<FY>/<seq>`, allocated inside the issuing transaction using a per-property counter row locked `FOR UPDATE`. Draft invoices SHALL NOT consume numbers.
- **SRS-BILL-6** State machine:
  ```
  DRAFT ──issue──► ISSUED ──payment(partial)──► PARTIALLY_PAID ──payment(full)──► PAID
    │                 │                              │
    │                 └──cancel(reason)──► CANCELLED ◄┘   (PAID → CANCELLED requires refund first)
    └──delete (drafts only)
  OVERDUE is a derived flag: status ∈ {ISSUED, PARTIALLY_PAID} AND due_date < today
  WRITTEN_OFF: Owner-only terminal state from ISSUED/PARTIALLY_PAID, reason required
  ```
- **SRS-BILL-7** An ISSUED or later invoice SHALL be immutable: no line item or amount mutation. Corrections SHALL use (a) cancel + reissue while unpaid, or (b) a credit note referencing the invoice.
- **SRS-BILL-8** Late fee: if `balance_due > 0` and `today > due_date + grace_days`, the system SHALL add one `LATE_FEE` item per period (fixed amount or percentage of overdue rent per the property rule) to the next invoice, never more than once per period, waivable by Owner with a reason.
- **SRS-BILL-9** Invoice PDF SHALL be generated on issue, stored immutably, and re-servable byte-identically; regeneration SHALL be allowed only for cosmetic template changes and SHALL keep the original.
- **SRS-BILL-10** Cycle preview SHALL surface warnings: missing meter reading, amount deviating >25% from the tenancy's previous invoice, tenancy in notice period, unpaid prior balance, tenancy starting/ending mid-period.
- **SRS-BILL-11** Auto-issue MAY be enabled per property; when disabled (default), invoices remain DRAFT until an owner or manager issues them.
- **SRS-BILL-12** Timezone: billing days, due dates and overdue evaluation SHALL use `Asia/Kolkata` calendar dates. A `billing_day` greater than the days in a month SHALL clamp to the last day (hence the 1–28 recommendation).

### 4.6 Payments (`payments`)
- **SRS-PAY-1** A payment SHALL record: amount, received_on (date), method ∈ {CASH, UPI, BANK_TRANSFER, CHEQUE, CARD, ADJUSTMENT}, reference, received_by (user), note, optional receipt image, tenancy.
- **SRS-PAY-2** A payment SHALL be allocated across invoices as one or more `payment_allocation` rows; default allocation is oldest-issued-first; the sum of allocations SHALL NOT exceed the payment amount; the unallocated remainder SHALL be held as tenancy credit.
- **SRS-PAY-3** Tenancy credit SHALL be auto-applied to the next issued invoice.
- **SRS-PAY-4** Over-allocation to an invoice beyond its balance SHALL be rejected `422 OVER_ALLOCATION`.
- **SRS-PAY-5** Payments SHALL NOT be edited. Corrections use a reversal (`ADJUSTMENT` with negative allocation effect) that is audit-logged.
- **SRS-PAY-6** A receipt PDF SHALL be produced per payment with a gapless per-property receipt number.
- **SRS-PAY-7** Dues ageing SHALL bucket each invoice's `balance_due` by `today − due_date` into 0–7, 8–15, 16–30, 30+ days.
- **SRS-PAY-8** The module SHALL expose `PaymentProvider { createCollectionRequest(), handleWebhook(), getStatus() }` with a `ManualProvider` implementation in v1; adding a gateway SHALL require no change to ledger or allocation logic.
- **SRS-PAY-9** Refunds SHALL be recorded as `refund` rows linked to a tenancy and (where applicable) a settlement, never as negative payments.

### 4.7 Utilities / electricity (`utilities`)
- **SRS-UTIL-1** A meter SHALL belong to a room or a bed and have an identifier, a unit rate source (property default or meter override) and an optional fixed monthly charge.
- **SRS-UTIL-2** A reading SHALL record: meter, period, previous_reading, current_reading, read_on, rate_paise_per_unit, optional photo, recorded_by. `current_reading < previous_reading` SHALL be rejected unless flagged as a meter reset with a reason.
- **SRS-UTIL-3** Metered electricity for a room in a period:
  ```
  units  = current − previous
  amount = units × rate + fixed_charge
  share(tenancy) = amount × occupied_days(tenancy) / Σ occupied_days(all tenancies in room)
  ```
  If no tenancy occupied the room in the period, the amount SHALL post as an owner expense instead of being billed.
- **SRS-UTIL-4** Flat mode SHALL bill a fixed amount per bed per period, pro-rated by occupied days. Included mode SHALL bill nothing.
- **SRS-UTIL-5** The system SHALL warn when `units` exceeds 3× the trailing 3-period average for that meter, and require confirmation to save.
- **SRS-UTIL-6** Rounding: the room amount SHALL be split so the sum of shares equals the room amount exactly, with any rounding remainder assigned to the largest share.

### 4.8 Expenses (`expenses`)
- **SRS-EXP-1** An expense SHALL record: property, category, amount, spent_on, paid_to, method, note, optional receipt file, created_by.
- **SRS-EXP-2** Categories SHALL be a fixed enum plus org-defined custom categories.
- **SRS-EXP-3** Recurring templates SHALL generate a reminder (not an expense) on the due day; the user confirms the actual amount.
- **SRS-EXP-4** Salary payments recorded in `employees` SHALL create a linked expense in category `STAFF_SALARY`; that expense SHALL NOT be independently editable (edit the salary payment instead).

### 4.9 Complaints (`complaints`)
- **SRS-CMP-1** A complaint SHALL record: property, room, bed (optional), tenancy (optional), category, description, priority, source ∈ {OWNER, STAFF, TENANT_LINK, PHONE, WHATSAPP}, media[], assignee (optional), due_by, status.
- **SRS-CMP-2** Status machine:
  ```
  PENDING ──assign/start──► IN_PROGRESS ──resolve(note)──► RESOLVED ──reopen(≤48h)──► IN_PROGRESS
     │                          │
     └────reject(reason)────────┴──► REJECTED
  ```
  Every transition SHALL append a `complaint_event` (actor, from, to, note, visibility ∈ {INTERNAL, TENANT_VISIBLE}, at).
- **SRS-CMP-3** `due_by` SHALL default from priority SLA (URGENT 4h, HIGH 24h, MEDIUM 72h, LOW 7d), configurable per org.
- **SRS-CMP-4** Resolution SHALL require a non-empty note. Rejection SHALL require a reason.
- **SRS-CMP-5** Tenant-visible transitions SHALL enqueue a notification to the tenant's phone with a tokenised status link.
- **SRS-CMP-6** The tokenised tenant link SHALL be a single-purpose, signed, expiring (30-day) URL granting read access to that complaint plus the ability to add a comment/photo and reopen within 48h of resolution. It SHALL NOT expose any other tenant, financial or personal data.
- **SRS-CMP-7** Media: up to 5 images (≤10 MB each) or 1 video (≤100 MB, ≤60 s) per complaint or comment.

### 4.10 Employees (`employees`)
- **SRS-EMP-1** Employee record per PRD-63; `monthly_salary_paise`, `salary_cycle_day`, `status` ∈ {ACTIVE, INACTIVE}.
- **SRS-EMP-2** Attendance SHALL store one record per (employee, date) with a status enum; bulk marking for a date SHALL be supported; future dates SHALL be rejected.
- **SRS-EMP-3** Payable salary for a month:
  ```
  payable = monthly_salary − (unpaid_leave_days + absent_days) × monthly_salary / days_in_month
            − advances_outstanding
  ```
  Rules SHALL be configurable per org; the computation SHALL be shown as a breakdown, never as a bare number.
- **SRS-EMP-4** Recording a salary payment SHALL post an expense per SRS-EXP-4 and reduce advances first if any.

### 4.11 Notices and messaging (`notices`, `messaging`)
- **SRS-MSG-1** `MessagingProvider { sendTemplate(to, template, vars, channel), getStatus(id) }` SHALL abstract the BSP, with implementations `whatsapp`, `sms`, and `noop` (for dev/test).
- **SRS-MSG-2** Every outbound message SHALL be persisted before dispatch with status ∈ {QUEUED, SENT, DELIVERED, READ, FAILED} and a provider message id; status callbacks SHALL update it.
- **SRS-MSG-3** Failed messages SHALL retry with exponential backoff up to 5 attempts, then surface in a failures list with a manual retry and a "copy message" fallback so the owner can send it by hand.
- **SRS-MSG-4** Audience resolution SHALL support: property, floors, rooms, selected tenancies, all with dues, all in notice period — evaluated at send time and snapshotted into recipients.
- **SRS-MSG-5** Rent reminder schedule SHALL be configurable per property as offsets in days relative to `due_date` (e.g. `[-3, 0, +3, +7, +15]`); a reminder SHALL be skipped if `balance_due = 0` at evaluation time, if reminders are paused for that tenancy, or if the same reminder offset was already sent for that invoice.
- **SRS-MSG-6** Quiet hours 21:00–08:00 IST SHALL suppress non-emergency sends until the window opens; EMERGENCY type SHALL bypass.
- **SRS-MSG-7** Templates SHALL be versioned; only variable substitution is editable in-app, because BSP templates require pre-approval.

### 4.12 Analytics and reports (`analytics`)
- **SRS-ANA-1** Dashboard metrics for a period SHALL be defined exactly as:
  | Metric | Definition |
  | --- | --- |
  | Billed revenue | Σ `total` of invoices issued with `period_start` in the range, excluding cancelled |
  | Collected | Σ payment allocations with `received_on` in the range |
  | Pending | Σ `balance_due` of non-cancelled invoices as of now |
  | Expenses | Σ expenses with `spent_on` in the range |
  | Net profit | Collected − Expenses (cash basis) **and** Billed − Expenses (accrual) shown as two clearly labelled figures |
  | Occupancy % | per SRS-PROP-5 |
- **SRS-ANA-2** Every metric SHALL be drillable to the underlying record list, and the drill-down total SHALL equal the metric exactly.
- **SRS-ANA-3** Reports: rent roll, collection, dues ageing, expense summary, occupancy, tenant register, P&L per property per month. All exportable to CSV; P&L and rent roll also to PDF.
- **SRS-ANA-4** Reports SHALL be computed from source tables (with optional materialised monthly rollups refreshed on relevant domain events); a stale rollup SHALL never be served as current-month data.

### 4.13 Subscription (`subscription`)
- **SRS-SUB-1** `managed_beds(org) = count of non-archived beds across all properties` (assumption A-5, pending OQ-2), recomputed on `BedCountChanged`.
- **SRS-SUB-2** Plan mapping: Starter ≤50, Growth ≤100, Enterprise >100 (custom). Trial 14 days.
- **SRS-SUB-3** At ≥90% of the plan limit the UI SHALL warn. Above the limit, bed creation SHALL be blocked (`402 PLAN_LIMIT_EXCEEDED`) and after a 7-day grace period invoice generation SHALL be blocked. Read access, payment recording and exports SHALL never be blocked.
- **SRS-SUB-4** States: `TRIALING → ACTIVE → PAST_DUE → CANCELLED`, with dunning notices at +1, +4, +7 days past due.
- **SRS-SUB-5** Org data export SHALL be available on demand in all states, including CANCELLED, for 90 days after cancellation.

### 4.14 Audit (`iam`)
- **SRS-AUD-1** The system SHALL append an audit entry for: authentication events, permission and user changes, tenancy lifecycle changes, rent revisions, invoice issue/cancel/write-off, payment and refund records, deposit changes, settlement, ID-proof reads, exports, impersonation, and any data deletion.
- **SRS-AUD-2** Entries SHALL record actor, org, entity type/id, action, before/after JSON diff, IP, user agent, timestamp; and SHALL be append-only (no UPDATE/DELETE grants for the application role).

---

## 5. Data requirements

### 5.1 Core schema (abbreviated DDL)
```sql
-- every domain table carries org_id and is RLS-scoped
create table organizations (id uuid pk, name text, city text, plan text,
  created_at timestamptz);

create table users (id uuid pk, org_id uuid fk, phone text, email text,
  password_hash text null, name text, role text check (role in
  ('OWNER','MANAGER','ACCOUNTANT','STAFF')), status text,
  unique (org_id, phone));

create table properties (id uuid pk, org_id uuid fk, name text, address jsonb,
  geo point null, billing_day smallint, electricity_mode text, food_mode text,
  late_fee_rule jsonb, notice_period_days int, invoice_prefix text,
  auto_issue boolean default false, reminder_offsets int[] default '{-3,0,3,7,15}');

create table rooms (id uuid pk, org_id uuid, property_id uuid, floor int,
  number text, sharing_capacity int, room_type text[], default_rent_paise bigint,
  amenities text[], archived_at timestamptz null, unique (property_id, number));

create table beds (id uuid pk, org_id uuid, room_id uuid, label text,
  rent_paise bigint null, archived_at timestamptz null, unique (room_id, label));

create table bed_blocks (id uuid pk, bed_id uuid, from_date date, to_date date null,
  reason text);

create table tenants (id uuid pk, org_id uuid, name text, phone text,
  alt_phone text, email text, occupation text, gender text, dob date,
  address jsonb, emergency_contact jsonb not null, deleted_at timestamptz null,
  unique (org_id, phone));

create table tenant_documents (id uuid pk, tenant_id uuid, kind text,
  number_encrypted bytea, storage_key text, uploaded_by uuid, uploaded_at timestamptz);

create table tenancies (id uuid pk, org_id uuid, property_id uuid, tenant_id uuid,
  bed_id uuid, move_in_date date, notice_date date null,
  expected_vacate_date date null, actual_vacate_date date null,
  agreed_rent_paise bigint, deposit_agreed_paise bigint,
  deposit_received_paise bigint default 0, billing_day smallint,
  notice_period_days int, food_included boolean, recurring_charges jsonb,
  status text check (status in ('ACTIVE','NOTICE_PERIOD','CLOSED','CANCELLED')),
  reminders_paused boolean default false);
-- no two overlapping active tenancies per bed
create index on tenancies (bed_id, move_in_date, actual_vacate_date);

create table rent_revisions (id uuid pk, tenancy_id uuid, amount_paise bigint,
  effective_from date, reason text, actor_id uuid, created_at timestamptz);

create table bed_assignments (id uuid pk, tenancy_id uuid, bed_id uuid,
  from_date date, to_date date null);  -- supports transfers within a tenancy

create table invoices (id uuid pk, org_id uuid, property_id uuid, tenancy_id uuid,
  number text null, period_start date, period_end date, issue_date date null,
  due_date date, total_paise bigint, status text, cancelled_reason text null,
  pdf_key text null, created_at timestamptz);
create unique index on invoices (tenancy_id, period_start)
  where status <> 'CANCELLED';
create unique index on invoices (property_id, number) where number is not null;

create table invoice_line_items (id uuid pk, invoice_id uuid, type text,
  description text, quantity numeric null, unit_rate_paise bigint null,
  amount_paise bigint, meta jsonb);

create table payments (id uuid pk, org_id uuid, tenancy_id uuid, amount_paise bigint,
  received_on date, method text, reference text, received_by uuid, note text,
  receipt_number text, receipt_key text null, reversed_by uuid null);

create table payment_allocations (id uuid pk, payment_id uuid, invoice_id uuid,
  amount_paise bigint);

create table refunds (id uuid pk, tenancy_id uuid, amount_paise bigint,
  paid_on date, method text, reference text, settlement_id uuid null);

create table settlements (id uuid pk, tenancy_id uuid, vacate_date date,
  breakdown jsonb, net_paise bigint, direction text, pdf_key text,
  created_by uuid, reversed_by uuid null);

create table meters (id uuid pk, org_id uuid, room_id uuid null, bed_id uuid null,
  identifier text, rate_paise_per_unit bigint null, fixed_charge_paise bigint default 0);

create table meter_readings (id uuid pk, meter_id uuid, period_start date,
  previous_reading numeric, current_reading numeric, rate_paise_per_unit bigint,
  read_on date, photo_key text null, recorded_by uuid,
  unique (meter_id, period_start));

create table expenses (id uuid pk, org_id uuid, property_id uuid, category text,
  amount_paise bigint, spent_on date, paid_to text, method text, note text,
  receipt_key text null, source text default 'MANUAL', source_id uuid null);

create table complaints (id uuid pk, org_id uuid, property_id uuid, room_id uuid null,
  bed_id uuid null, tenancy_id uuid null, category text, description text,
  priority text, source text, assignee_id uuid null, due_by timestamptz,
  status text, resolved_at timestamptz null);

create table complaint_events (id uuid pk, complaint_id uuid, actor_id uuid null,
  from_status text, to_status text, note text, visibility text, media jsonb,
  created_at timestamptz);

create table employees (id uuid pk, org_id uuid, property_id uuid, name text,
  role text, phone text, monthly_salary_paise bigint, joining_date date,
  emergency_contact jsonb, status text);

create table attendance (id uuid pk, employee_id uuid, on_date date, status text,
  marked_by uuid, unique (employee_id, on_date));

create table salary_payments (id uuid pk, employee_id uuid, period_month date,
  amount_paise bigint, paid_on date, method text, breakdown jsonb, expense_id uuid);

create table notices (id uuid pk, org_id uuid, property_id uuid, type text,
  title text, body text, attachment_key text null, audience jsonb,
  created_by uuid, sent_at timestamptz null);

create table messages (id uuid pk, org_id uuid, notice_id uuid null,
  tenancy_id uuid null, to_phone text, channel text, template text, vars jsonb,
  status text, provider_message_id text null, attempts int default 0,
  last_error text null, created_at timestamptz, delivered_at timestamptz null);

create table audit_log (id bigserial pk, org_id uuid, actor_id uuid null,
  entity_type text, entity_id uuid, action text, before jsonb, after jsonb,
  ip inet, user_agent text, created_at timestamptz default now());

create table subscriptions (id uuid pk, org_id uuid, plan text, bed_count int,
  status text, trial_ends_on date, current_period_end date);

create table outbox (id bigserial pk, org_id uuid, event_type text,
  payload jsonb, processed_at timestamptz null);
```

### 5.2 Integrity rules
- **SRS-DATA-1** Overlapping active tenancies per bed SHALL be prevented by a database-level exclusion constraint on the date range, not application code alone.
- **SRS-DATA-2** Monetary columns SHALL be `bigint` paise; no `float`/`double` anywhere in the schema.
- **SRS-DATA-3** Foreign keys SHALL be enforced; deletions SHALL be restricted where history matters (invoices, payments, audit).
- **SRS-DATA-4** Row-level security SHALL restrict every domain table by `org_id` from the session context.
- **SRS-DATA-5** Backups: nightly full + point-in-time recovery to any moment in the last 7 days; restore SHALL be exercised before pilot launch and quarterly after.
- **SRS-DATA-6** Retention: financial records retained 8 years (Indian bookkeeping norms); complaint media 2 years; message logs 1 year; ID proofs deleted 90 days after tenancy close unless legally required, on owner confirmation.

---

## 6. Interface requirements

### 6.1 API conventions
- REST over HTTPS, JSON, base `/api/v1`. Auth via `Authorization: Bearer` or httpOnly cookie.
- Resource-oriented paths, cursor pagination (`?cursor=&limit=`), `X-Request-Id` on every response.
- Mutating endpoints accept `Idempotency-Key`; replay returns the original result.
- Errors: `{ "error": { "code": "BED_ALREADY_OCCUPIED", "message": "...", "details": {...} } }` with stable machine codes.
- OpenAPI spec generated from code and published.

### 6.2 Representative endpoints
```
POST   /auth/otp/request                 {phone}
POST   /auth/otp/verify                  {phone, code} → tokens, memberships
GET    /properties                       list
POST   /properties/:id/rooms/bulk        {floor, from, to, sharing, rent, tags}
GET    /properties/:id/occupancy?on=     grid projection
POST   /tenancies                        create (move-in)
POST   /tenancies/:id/rent-revisions     {amount, effective_from, reason}
POST   /tenancies/:id/transfer           {bed_id, transfer_date}
POST   /tenancies/:id/notice             {notice_date}
POST   /tenancies/:id/settlement         {vacate_date, deductions[]} → settlement
POST   /billing/cycles                   {property_id, period} → cycle_run (drafts)
GET    /billing/cycles/:id               totals + rows + warnings
POST   /billing/cycles/:id/issue         bulk issue
POST   /invoices/:id/cancel              {reason}
GET    /invoices/:id/pdf                 signed URL
POST   /payments                         {tenancy_id, amount, method, allocations[]}
GET    /dues?property_id=&bucket=        ageing list
POST   /meters/:id/readings              {period, previous, current, rate, photo}
POST   /expenses                         create
GET    /analytics/dashboard?property=&period=
GET    /reports/rent-roll.csv
POST   /complaints                       create
POST   /complaints/:id/transitions       {to, note, visibility, assignee_id}
POST   /notices                          {type, title, body, audience} → dispatch
GET    /public/complaints/:token         tokenised tenant view (no auth)
```

### 6.3 External interfaces
- **BSP** (WhatsApp/SMS): template send + delivery webhooks; secret-verified callbacks; provider-agnostic adapter.
- **Object storage**: S3 API; private buckets; 15-minute signed URLs; server-side upload validation of MIME and size.
- **PDF**: internal renderer service; templates versioned with the invoice so an old invoice re-renders identically.
- **Payment gateway (future)**: `PaymentProvider` per SRS-PAY-8; webhook handler idempotent on gateway event id.

### 6.4 UI requirements
- **SRS-UI-1** Screens: login/OTP, setup wizard, dashboard, occupancy grid, rooms, tenants list/detail, tenancy create/settle, rent cycle preview, invoice detail, dues, record payment, meter readings, expenses, complaints board, employees/attendance, notices composer, reports, settings, users, subscription.
- **SRS-UI-2** Occupancy grid, dues list, record payment, meter reading entry and complaint update SHALL be fully operable on a 360×640 viewport with one hand.
- **SRS-UI-3** Money SHALL always display as ₹ with Indian digit grouping (₹1,23,456) and never expose paise arithmetic to the user.
- **SRS-UI-4** Every list SHALL support search and the filters named in the PRD, with filter state reflected in the URL.
- **SRS-UI-5** Forms SHALL validate inline, preserve input on failure (SRS-UI-6) and disable submit only while a request is in flight.
- **SRS-UI-6** Meter reading and payment forms SHALL persist their draft locally and offer retry after a network failure.
- **SRS-UI-7** Empty states SHALL tell the user the next action, not just "no data".

---

## 7. Non-functional requirements

### 7.1 Performance
- **SRS-NFR-1** p95 read API < 400 ms, p95 write < 1200 ms at 200 concurrent orgs.
- **SRS-NFR-2** Dashboard first contentful paint < 2.5 s on a simulated 3G phone; occupancy grid for 200 beds renders < 1 s.
- **SRS-NFR-3** A cycle run for 500 tenancies completes < 60 s; PDF generation is asynchronous and batched.
- **SRS-NFR-4** Reports up to 12 months of data return < 3 s; larger exports run as background jobs with a download link.

### 7.2 Security
- **SRS-NFR-5** Authorization SHALL be enforced server-side per endpoint AND per row (org scope); the UI hiding a control is never the control.
- **SRS-NFR-6** Role matrix (deny by default):
  | Capability | Owner | Manager | Accountant | Staff |
  | --- | --- | --- | --- | --- |
  | View profit / P&L | ✔ | ✘ | ✔ | ✘ |
  | Issue / cancel invoice | ✔ | ✔ | ✘ | ✘ |
  | Write off / waive late fee | ✔ | ✘ | ✘ | ✘ |
  | Record payment / refund | ✔ | ✔ | ✘ | ✘ |
  | Create / close tenancy | ✔ | ✔ | ✘ | ✘ |
  | View full ID proof | ✔ | ✘ | ✘ | ✘ |
  | Enter expenses | ✔ | ✔ | ✔ | ✘ |
  | Meter readings | ✔ | ✔ | ✘ | ✔ |
  | Complaints (all) | ✔ | ✔ | ✘ | assigned only |
  | Manage users / subscription | ✔ | ✘ | ✘ | ✘ |
  | Export data | ✔ | ✘ | ✔ | ✘ |
- **SRS-NFR-7** Transport TLS 1.2+; secrets in a managed secret store, never in the repo; at-rest encryption for the database and buckets.
- **SRS-NFR-8** OWASP Top 10 controls: parameterised queries only, output encoding, CSRF protection on cookie auth, strict CORS allowlist, security headers/CSP, per-IP and per-user rate limits, file-type and size validation, no user input in shell or path operations.
- **SRS-NFR-9** Tokenised public links SHALL be signed, single-purpose, expiring, revocable, and rate-limited.
- **SRS-NFR-10** DPDP posture: purpose-limited collection, consent record for tenant data, deletion/export request handling within 30 days, processor agreements with BSP and hosting.
- **SRS-NFR-11** Dependency and container scanning in CI; no new dependency version younger than 7 days.

### 7.3 Reliability and operations
- **SRS-NFR-12** Availability target 99.5% monthly; planned maintenance outside 08:00–22:00 IST.
- **SRS-NFR-13** RPO ≤ 15 minutes, RTO ≤ 4 hours.
- **SRS-NFR-14** All jobs idempotent and retried with backoff; a dead-letter queue with an alert.
- **SRS-NFR-15** Alerts SHALL fire on: failed cycle run, PDF backlog, message failure rate >5%, error rate >1%, job queue depth, DB connection saturation.
- **SRS-NFR-16** Structured logs with request id, org id, user id; no PII (phone, ID numbers) in logs.
- **SRS-NFR-17** Zero-downtime deploys; migrations backwards-compatible (expand/contract).

### 7.4 Maintainability and quality gates
- **SRS-NFR-18** TypeScript strict mode; no `any`; lint and typecheck clean in CI.
- **SRS-NFR-19** Test coverage: billing, payments, utilities, tenancy settlement modules ≥ 90% line and branch; overall ≥ 70%.
- **SRS-NFR-20** Golden-file tests for invoice PDFs and settlement statements.
- **SRS-NFR-21** Property-based tests for pro-ration, allocation and electricity splitting (invariants: shares sum to total; allocations never exceed payment; balance never negative).
- **SRS-NFR-22** Every SRS requirement referenced by at least one automated test (traceability report in CI).
- **SRS-NFR-23** Seed script producing a realistic 3-property, 120-bed, 6-month-history org for development and demos.

### 7.5 Usability and accessibility
- **SRS-NFR-24** A new owner SHALL be able to reach "first invoice issued" without support, guided only by the setup wizard and checklist.
- **SRS-NFR-25** WCAG 2.1 AA for colour contrast, focus visibility, labels and keyboard operability of all forms.
- **SRS-NFR-26** All copy externalised for localisation; no concatenated sentences in code.

---

## 8. Validation

### 8.1 Critical test scenarios (must pass before pilot)
| # | Scenario | Expected |
| --- | --- | --- |
| T1 | Mid-month move-in on the 12th of a 30-day month, rent ₹6,000 | First invoice rent line = ₹3,800 (19 days), day count printed |
| T2 | Cycle run executed twice for the same period | Exactly one invoice per tenancy; second run reports "already generated" |
| T3 | Rent revised from ₹6,000 to ₹7,000 effective the 16th of a 30-day month | Rent line = ₹3,000 + ₹3,666.67→₹3,667 with both segments shown |
| T4 | Room of 3 beds, 2 occupied, 200 units at ₹8, one tenant occupied 15 of 30 days | Amount ₹1,600 split 2:1 by occupied days; shares sum to exactly ₹1,600 |
| T5 | Payment of ₹5,000 against invoices of ₹3,000 (older) and ₹4,000 | ₹3,000 + ₹2,000 allocated; second invoice PARTIALLY_PAID |
| T6 | Payment of ₹10,000 against ₹6,000 dues | ₹6,000 allocated, ₹4,000 tenancy credit; auto-applied next cycle |
| T7 | Move-out with deposit ₹10,000, final rent ₹2,000, dues ₹3,000, damages ₹1,500 | Refund due ₹3,500; statement itemises all four |
| T8 | Move-out where dues exceed deposit | Net receivable, never a negative refund |
| T9 | Attempt to reduce sharing capacity with an occupied bed | 409 with the blocking tenant named |
| T10 | Attempt to edit an ISSUED invoice | Rejected; cancel+reissue path offered |
| T11 | Manager attempts to view P&L or manage users | 403, and the nav item is absent |
| T12 | Two overlapping tenancies created concurrently on one bed | Exactly one succeeds (DB constraint) |
| T13 | Reminder job runs after full payment | No reminder sent |
| T14 | BSP outage during a cycle send | Messages QUEUED then retried; owner sees the failure list with manual-copy fallback |
| T15 | Missing meter reading at cycle time | Invoice issues without an electricity line, warning shown, reading can be billed next cycle |
| T16 | Bed count crosses 50 on the Starter plan | Upgrade prompt; new bed creation blocked; existing operations unaffected |
| T17 | Dashboard drill-down on "Collected" | Sum of listed payments equals the metric exactly |
| T18 | Bed transfer on the 10th | Invoice shows both beds pro-rated; history shows the transfer |

### 8.2 Acceptance for the release
All P0 requirements implemented, T1–T18 automated and passing, coverage gates met, a restore drill completed, and two consecutive live rent cycles run at three pilot PGs with no billing dispute traceable to the software.

---

## 9. Traceability
Each SRS requirement maps to PRD items and tests: `SRS-BILL-*` ← PRD-25…35, tests T1–T3, T10, T15; `SRS-PAY-*` ← PRD-36…42, tests T5, T6, T17; `SRS-UTIL-*` ← PRD-43…47, test T4; `SRS-TCY-*` ← PRD-14…24, tests T7–T9, T12, T18; `SRS-NFR-6` ← PRD §2, test T11. A generated traceability matrix is a CI artifact (SRS-NFR-22).

## 10. Open items
Inherited from PRD §8 (OQ-1…OQ-9). Additionally:
- **SRS-OQ-1** Are per-floor (not per-room) electricity meters in use at pilot PGs? That requires a floor-level split rule not specified here.
- **SRS-OQ-2** Is a configurable tax/GST line needed on invoices at launch?
- **SRS-OQ-3** Preferred hosting (AWS Mumbai vs. a managed PaaS) — affects RLS/KMS specifics and cost.
