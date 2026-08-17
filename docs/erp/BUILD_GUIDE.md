# NEXSTAY PG Management ERP — Step-by-Step Build Guide

Companion to [PRD.md](PRD.md) (what to build) and [SRS.md](SRS.md) (exact behaviour). This document is the execution plan: environment, repo layout, ordered milestones with concrete tasks, and the definition of done for each.

Effort is given in **build sessions** (one focused Devin session ≈ one to two weeks of a single developer). Sequence matters more than the estimates.

---

## 0. Prerequisites and decisions

### 0.1 Accounts and services to arrange
| Need | Suggested | Blocking from |
| --- | --- | --- |
| Hosting | AWS Mumbai (ap-south-1) or Railway/Render with an Indian region | Step 1 (can develop locally first) |
| Postgres | Managed Postgres 15+ (RDS/Neon/Railway) | Step 1 |
| Redis | Managed Redis 7+ | Step 4 |
| Object storage | Cloudflare R2 or S3 (private buckets) | Step 3 |
| WhatsApp/SMS | MSG91 or Gupshup (WhatsApp Business API + SMS) — template approval takes days, start early | Step 8 |
| Error tracking | Sentry free tier | Step 1 |
| Domain + email | e.g. `nexstay.in`, transactional email later | Step 10 |

Start the WhatsApp BSP application on day one: template approval is the longest external lead time in the whole plan.

### 0.2 Decisions needed before coding
Answer these (from PRD §8) — each one changes code:
1. **OQ-1** Record-only payments in v1? *Recommended: yes.* Ship faster, avoid settlement regulation, keep `PaymentProvider` pluggable.
2. **OQ-3** Late-fee default (suggest: ₹100 flat after 5 grace days, owner-configurable).
3. **OQ-4** Staff logins in v1? *Recommended: no* — Manager role only, staff work is entered by the manager. Saves a full permission surface.
4. **OQ-5/SRS-OQ-2** GST line on invoices at launch?
5. **OQ-8/SRS-OQ-1** Are pilot meters per room, per bed, or per floor?
6. **OQ-9** Food included in rent, or billed separately?
7. **OQ-7** Existing wireframes to match?

If any answer is unknown, build the assumed default from the PRD and keep it behind a property-level setting.

### 0.3 Ground rules for the whole build
1. Money is `bigint` paise. No floats. Ever.
2. Cross-check every financial feature against the SRS test table (§8.1) before calling it done.
3. Every list endpoint is org-scoped, paginated and filtered server-side from the first commit.
4. Ship vertical slices: schema → API → UI → test for one feature, not all schemas then all APIs.
5. Put a real pilot owner in front of it from milestone M1 onward; their Excel sheet is the acceptance test.

---

## 1. Step 1 — Repository, tooling, CI (0.5 session)

### Tasks
1. Initialise a pnpm workspace monorepo:
```
nexstay/
├─ apps/
│  ├─ api/            # NestJS
│  └─ web/            # Next.js (App Router)
├─ packages/
│  ├─ shared/         # zod schemas, DTO types, money & date utils, error codes
│  ├─ db/             # Prisma schema, migrations, seed
│  └─ config/         # eslint, tsconfig, prettier presets
├─ docs/
└─ docker-compose.yml # postgres + redis for local dev
```
2. TypeScript strict everywhere; ESLint + Prettier; `pnpm lint`, `pnpm typecheck`, `pnpm test`, `pnpm build` scripts at the root.
3. Money utilities in `packages/shared`: `Paise` branded type, `toPaise`, `formatINR` (Indian grouping), `prorate(amountPaise, days, daysInPeriod)`, `splitProportionally(amount, weights[])` guaranteeing the shares sum exactly. Unit-test these first — they underpin everything.
4. Date utilities pinned to `Asia/Kolkata`: `periodFor(billingDay, month)`, `daysInPeriod`, `overlapDays(range, range)`.
5. GitHub Actions CI: install → lint → typecheck → unit tests → build; Postgres service container for integration tests; branch protection on `main`.
6. Sentry, structured logging (pino), `X-Request-Id` middleware.
7. `docker-compose up` gives a working local Postgres + Redis; `pnpm dev` runs API and web together.
8. Seed script skeleton (SRS-NFR-23).

### Done when
CI is green on an empty-but-wired app, `pnpm dev` serves both apps locally, and the money/date utilities have 100% test coverage.

---

## 2. Step 2 — Auth, orgs, roles, audit (1 session)

### Tasks
1. Schema: `organizations`, `users`, `sessions`, `audit_log` (SRS §5.1).
2. OTP flow (SRS-AUTH-1…3): request/verify endpoints, 6-digit code, 5-minute expiry, single-use, rate limits, lockout. In dev, log the OTP instead of sending it; wire the real channel in Step 8.
3. JWT access (15 min) + rotating refresh (30 days) in httpOnly cookies; device list and revocation; CSRF protection.
4. Org membership and role model; `@RequirePermission()` guard driven by the SRS-NFR-6 matrix as data, not scattered `if` statements.
5. Postgres RLS: session variable `app.current_org`, policies on every domain table; a test that proves cross-org access fails at the DB even if the app layer is bypassed.
6. Audit interceptor writing before/after diffs for annotated mutations (SRS-AUD-1/2); revoke UPDATE/DELETE on `audit_log` for the app role.
7. Web: OTP login screen, org selection when multiple memberships, session refresh, protected layout, role-aware navigation.
8. User invite + deactivate (SRS-AUTH-6/7).

### Done when
An owner can sign up by OTP, invite a manager, the manager's forbidden endpoints return 403, cross-org reads fail at the database, and every mutation appears in the audit log.

---

## 3. Step 3 — Properties, rooms, beds, occupancy (1 session)

### Tasks
1. Schema: `properties`, `rooms`, `beds`, `bed_blocks`; property settings (billing day, electricity mode, food mode, late-fee rule, notice period, invoice prefix, auto-issue, reminder offsets).
2. Bulk room creation (SRS-PROP-2) and capacity change rules (SRS-PROP-3).
3. Occupancy derivation (SRS-PROP-4) as a pure function over tenancies + blocks, plus the `bed_occupancy_current` projection and a `rebuild-occupancy` command. Write the pure function first and test it against hand-computed cases.
4. Documents module: signed upload URLs, MIME/size validation, private buckets, 15-minute signed reads.
5. Web: property switcher, setup wizard steps 1–2 (property, floors/rooms in bulk), occupancy grid (colour-coded, mobile-first, tap a bed for actions), vacancy list with days-vacant.

### Done when
A 50-bed PG can be set up in under 10 minutes on a phone, the occupancy grid renders 200 beds in under a second, and reducing capacity below occupancy is refused with the blocking tenant named.

---

## 4. Step 4 — Tenants and tenancies (1 session)

### Tasks
1. Schema: `tenants`, `tenant_documents`, `tenancies`, `rent_revisions`, `bed_assignments`.
2. Field-level encryption for ID numbers (SRS-TEN-3), masked reads, audited full reads.
3. Tenancy creation with the overlap exclusion constraint (SRS-DATA-1) — write a concurrency test that fires two creations at once (T12).
4. Rent revisions (SRS-TCY-5) and bed transfer (SRS-TCY-6) with `bed_assignments`.
5. Notice recording and expected-vacate computation (SRS-TCY-7).
6. CSV tenant import with per-row validation and an error report (SRS-TEN-5) — this is what gets pilot owners off Excel in an hour instead of a week.
7. Redis + BullMQ set up here (needed next step); queue dashboard in dev.
8. Web: tenants list with search, tenant detail timeline, tenancy create flow from a bed, notice action, ID proof upload, import screen with preview.

### Done when
A pilot PG's entire existing tenant sheet imports cleanly, tenancies show correct occupancy, and T9/T12 pass.

---

## 5. Step 5 — Billing engine (1.5 sessions) — the highest-risk step

Build this as a pure domain module first, with no HTTP and no database in the tests.

### Tasks
1. `packages/shared` or `api/src/billing/domain`: pure functions
   - `buildInvoice(tenancy, period, revisions, readings, charges, settings) → LineItem[]`
   - `prorateRent(revisions, occupiedRanges, period)`
   - `applyLateFee(previousInvoices, rule, today)`
   Unit-test against SRS §8.1 T1, T3 and a table of hand-computed cases before touching persistence.
2. Schema: `invoices`, `invoice_line_items`, invoice number counters; unique indexes from SRS-BILL-1 and SRS-BILL-5.
3. Cycle run: BullMQ repeatable job at 06:00 IST → for each due tenancy create a DRAFT invoice, idempotently (T2). Cycle-run record with totals and warnings (SRS-BILL-10).
4. Issue flow: allocate gapless numbers inside a transaction with `FOR UPDATE`, set due date, enqueue PDF, emit `InvoiceIssued`.
5. Immutability enforcement (SRS-BILL-7): cancel-and-reissue and credit notes; reject edits (T10).
6. Invoice PDF: HTML template + Puppeteer, versioned template, stored key, golden-file test (SRS-NFR-20). Include units and rate for electricity, day counts for pro-ration, and the late-fee policy line — these are the lines that prevent disputes.
7. Ad-hoc invoices; write-off (Owner only).
8. Web: rent cycle preview screen (totals, rows, warnings, bulk edit, **Issue all**), invoice detail, PDF download, bulk ZIP download.

### Done when
T1, T2, T3, T10, T15 pass; a 500-tenancy cycle run completes under 60 seconds; issuing twice never double-numbers; the PDF matches its golden file byte-for-byte.

---

## 6. Step 6 — Payments, dues, reminders-ready state (1 session)

### Tasks
1. Schema: `payments`, `payment_allocations`, `refunds`, receipt counters.
2. Allocation engine as a pure function: oldest-first default, manual override, over-allocation rejection, remainder → tenancy credit, auto-apply credit on next issue (T5, T6). Property-based tests for the invariants (SRS-NFR-21).
3. Payment reversal via compensating entries (SRS-PAY-5); refunds as first-class rows (SRS-PAY-9).
4. Receipt PDF with gapless numbering.
5. Dues ageing query (SRS-PAY-7) and collection report.
6. `PaymentProvider` interface with `ManualProvider` only (SRS-PAY-8) — an explicit seam so a gateway is a later addition, not a refactor.
7. Web: record-payment sheet (fast, phone-first: amount, date, method, allocate), dues list with ageing tabs and per-row remind button, tenancy ledger view.

### Done when
T5, T6, T17 pass; recording a payment from a phone takes under 15 seconds; dues totals reconcile exactly with invoice balances.

---

## 7. Step 7 — Electricity, expenses (0.5 session)

### Tasks
1. Schema: `meters`, `meter_readings`, `expenses`.
2. Electricity computation as a pure function with exact-sum splitting (SRS-UTIL-3/6) — T4 including the rounding-remainder case.
3. Reading entry screen optimised for bulk phone entry, showing last period's reading, implausible-jump warning (SRS-UTIL-5), optional meter photo.
4. Flat and included modes; missing-reading behaviour (SRS-UTIL-4, T15).
5. Expenses CRUD with categories, receipt upload, recurring templates producing reminders, monthly summary by category.

### Done when
T4 and T15 pass, and a manager can enter 30 meter readings on a phone in under 5 minutes.

---

## 8. Step 8 — Messaging, notices, automated reminders (1 session)

Start BSP template approval before this step; build against the `noop`/SMS provider meanwhile.

### Tasks
1. `MessagingProvider` abstraction with `whatsapp`, `sms`, `noop` implementations; persisted `messages` rows before dispatch; delivery webhooks; retry with backoff and a dead-letter list (SRS-MSG-1…3).
2. Templates (versioned, variables only): rent reminder, invoice issued, payment received, complaint update, notice, emergency, welcome.
3. Notice composer with audience resolution and recipient snapshotting (SRS-MSG-4).
4. Automated reminder scheduler from `reminder_offsets`, skipping paid/paused/already-sent (SRS-MSG-5, T13); quiet hours with emergency bypass (SRS-MSG-6).
5. Failure list with manual retry and a "copy message text" fallback so an owner is never blocked by a provider outage (T14).
6. Wire OTP delivery to the real SMS channel here.
7. Web: notices composer, message log with statuses, reminder settings per property.

### Done when
T13 and T14 pass, invoice issue triggers a real WhatsApp message to a test number, and reminders fire on schedule against a seeded overdue invoice.

---

## 9. Step 9 — Move-out settlement (0.5 session)

### Tasks
1. Settlement computation exactly per SRS-TCY-8 as a pure function, with T7 and T8 as tests.
2. Deductions entry (damage, cleaning, notice shortfall), final reading capture, deposit handling.
3. Immutable settlement record + statement PDF (golden-file test); Owner-only reversal via compensating record (SRS-TCY-9).
4. Bed freed on the actual vacate date (SRS-TCY-10); tenancy closed; history retained.
5. Web: settlement wizard showing every component and the net direction in plain language.

### Done when
T7, T8 pass; the deposit always appears explicitly; dues exceeding the deposit produce a receivable, never a negative refund.

---

## 10. Step 10 — Dashboard, reports, exports (1 session)

### Tasks
1. Metric definitions implemented exactly as SRS-ANA-1, with both cash-basis and accrual profit labelled distinctly.
2. Drill-down endpoints so every tile links to its underlying records, totals matching exactly (SRS-ANA-2, T17).
3. Reports: rent roll, collection, dues ageing, expense summary, occupancy, tenant register, monthly P&L. CSV export for all; PDF for P&L and rent roll.
4. Optional monthly rollup tables refreshed on domain events, never served stale for the current month (SRS-ANA-4).
5. 12-month trend charts; multi-property comparison.
6. Web: dashboard (mobile-first cards, then charts), reports index with date-range and property filters, background export with download link.

### Done when
Every tile drills down to matching records, and a pilot owner can answer the five questions in PRD §1.4 without asking anyone.

---

## 11. Step 11 — Complaints (0.5 session)

### Tasks
1. Schema `complaints`, `complaint_events`; state machine per SRS-CMP-2 with events for every transition.
2. Priority SLAs and `due_by`; assignment; internal vs. tenant-visible notes.
3. Media upload limits (SRS-CMP-7).
4. Tokenised tenant link (SRS-CMP-6): signed, expiring, single-purpose, rate-limited; view status, add a comment/photo, reopen within 48h.
5. Notifications on tenant-visible transitions (SRS-CMP-5).
6. Web: complaints board (columns by status) on desktop, list on mobile; complaint detail with timeline; public status page.

### Done when
A complaint can be raised from a WhatsApp link by a tenant with no account, the owner resolves it, and the tenant sees the update — with no other data exposed on the public link.

---

## 12. Step 12 — Employees, attendance, salary (0.5 session)

### Tasks
1. Schema `employees`, `attendance`, `salary_payments`.
2. Attendance bulk marking, monthly grid, future-date rejection.
3. Salary computation with a visible breakdown (SRS-EMP-3), advances, salary payment → auto-posted expense (SRS-EXP-4).
4. Complaint assignment to employees; simple work log.
5. Web: employees list/detail, attendance grid, salary run screen.

### Done when
Marking attendance for 8 staff takes under a minute, and a salary payment appears exactly once in expenses.

---

## 13. Step 13 — Subscription, plan limits, account settings (0.5 session)

### Tasks
1. Schema `subscriptions`; bed metering on `BedCountChanged` (SRS-SUB-1).
2. Plan mapping, 14-day trial, 90% warning, over-limit blocking with a 7-day grace, never blocking reads/payments/exports (SRS-SUB-3).
3. States and dunning notices (SRS-SUB-4).
4. Org data export (SRS-SUB-5) — also the DPDP answer.
5. Web: subscription page (plan, bed usage bar, invoices), upgrade request flow (manual/sales-assisted is fine at this stage).

### Done when
T16 passes and an owner can see exactly why they are being asked to upgrade.

---

## 14. Step 14 — Hardening, pilot readiness (1 session)

### Tasks
1. Performance pass against SRS-NFR-1…4 with a seeded 3-property/120-bed/6-month org; add missing indexes from `EXPLAIN` output.
2. Security pass: role matrix tests for every endpoint (T11), rate limits, security headers/CSP, dependency and container scans, secret hygiene, PII scrubbing in logs.
3. Reliability: alerts per SRS-NFR-15, dead-letter monitoring, **a real backup restore drill** (SRS-NFR-13) — do this before the pilot, not after an incident.
4. Traceability report in CI (SRS-NFR-22); coverage gates enforced.
5. Zero-downtime migration process documented (expand/contract).
6. Onboarding assets: setup wizard polish, checklist, empty states, a one-page owner guide in Hindi and English.
7. Staging environment with anonymised data; pilot onboarding runbook.

### Done when
All P0 requirements shipped, T1–T18 automated and green, a restore drill has succeeded, and three pilot PGs are onboarded.

---

## 15. Timeline summary

| Step | Content | Sessions | Cumulative |
| --- | --- | --- | --- |
| 1 | Repo, tooling, CI, money/date utils | 0.5 | 0.5 |
| 2 | Auth, orgs, roles, RLS, audit | 1 | 1.5 |
| 3 | Properties, rooms, beds, occupancy | 1 | 2.5 |
| 4 | Tenants, tenancies, import | 1 | 3.5 |
| 5 | Billing engine + invoice PDF | 1.5 | 5 |
| 6 | Payments, dues | 1 | 6 |
| 7 | Electricity, expenses | 0.5 | 6.5 |
| 8 | Messaging, notices, reminders | 1 | 7.5 |
| 9 | Move-out settlement | 0.5 | 8 |
| 10 | Dashboard, reports | 1 | 9 |
| 11 | Complaints | 0.5 | 9.5 |
| 12 | Employees | 0.5 | 10 |
| 13 | Subscription | 0.5 | 10.5 |
| 14 | Hardening, pilot readiness | 1 | 11.5 |

**Milestones**
- **M1 — Internal demo (after Step 5):** set up a PG, add tenants, generate and issue invoices with PDFs. Enough to show anyone what the product is.
- **M2 — Pilot-ready (after Step 8):** full rent cycle with payments, dues, electricity, expenses and automated WhatsApp reminders. Put it in a real PG here.
- **M3 — Complete ERP (after Step 13):** settlements, reports, complaints, employees, subscription. Chargeable.
- **M4 — Production-hardened (after Step 14):** performance, security, backups, onboarding.

Steps 1–8 are strictly sequential (each depends on the previous). Steps 9–13 are largely independent and can be reordered or parallelised across sessions; complaints (11) can be pulled earlier if pilot owners demand it, since it has no billing dependency.

---

## 16. Risks and mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Billing edge cases (pro-ration, transfers, mid-period revisions) produce a wrong bill | Destroys owner trust instantly | Pure-function domain layer, property-based tests, SRS §8.1 as a gate, cycle preview before issuing |
| WhatsApp template approval delayed | Reminders unavailable at pilot | Apply on day one; SMS fallback; manual "copy text" path |
| Owner won't leave Excel | No pilot data, no validation | CSV import in Step 4; 10-minute setup; keep the owner in the loop from M1 |
| Real electricity practices differ (per-floor meters, custom splits) | Rework in the billing core | Confirm OQ-8 before Step 7; keep the split rule pluggable per property |
| Scope creep toward marketplace/tenant app | Nothing ships | PRD §6 guardrails; module seams make later work additive |
| Payment gateway decision reversed late | Ledger rework | `PaymentProvider` seam from Step 6; ledger never assumes a provider |
| Multi-tenant data leak | Existential | RLS plus app scoping, with a test that bypasses the app layer |
| Single developer bus factor | Stalls | Documented decisions (these three docs), conventional commits, CI gates |

---

## 17. Working practices
- Branch per feature, PR into `main`, CI must be green; conventional commit messages.
- Every PR states which PRD/SRS requirement it implements and which tests cover it.
- Weekly: run the seeded demo end-to-end yourself; monthly: a restore drill and a dependency update pass.
- Keep these three documents current — when reality diverges from the spec, update the spec in the same PR.
