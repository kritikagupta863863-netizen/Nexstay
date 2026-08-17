# NEXSTAY PG Management ERP — Product Requirements Document

Status: Draft v1 — **scope: owner-side ERP only**
Owner: Shikhar Agnihotri
Sources: `docs/source-documents/`
Companion docs: [SRS.md](SRS.md) (formal spec), [BUILD_GUIDE.md](BUILD_GUIDE.md) (how to build it)
Wider platform context (marketplace + tenant app, out of scope here): [../PRD.md](../PRD.md)

---

## 1. Scope

### 1.1 In scope
A web application (mobile-responsive) used by a PG owner and their staff to run the PG business:
rooms and beds, tenants and tenancies, rent invoicing and collection, electricity billing, expenses,
complaints, employees, notices to tenants, business analytics, and multi-property support.

### 1.2 Explicitly out of scope for this release
- Public marketplace, PG discovery, reels/video feed, AI matching, roommate compatibility.
- Tenant-facing mobile app. **Tenants do not log in.** They receive WhatsApp/SMS messages and PDF invoices; everything is entered by the owner or staff.
- Online payment collection through a gateway. Payments are **recorded**, not processed — see section 5.4 and OQ-1.
- AI profit suggestions (needs historical data; revisit after 3–6 months of live use).
- Legal e-sign agreements, smart locks, embedded finance.

### 1.3 Why this order
The ERP is what owners pay for and the only surface that produces trustworthy data (occupancy, real rents, actual expenses). Everything else in the wider platform — marketplace listings, AI suggestions, tenant app — consumes ERP data, so it must exist first.

### 1.4 Definition of success for this release
1. Three real pilot PGs run **two consecutive full rent cycles** entirely in NEXSTAY, with no parallel Excel sheet.
2. Owner can answer, in under 10 seconds each: who hasn't paid, how much is pending, what's my occupancy, what did I spend this month, what's my profit.
3. Every rent reminder that used to be a manual WhatsApp message is sent by the system.
4. Zero billing disputes caused by the software (a dispute traceable to a wrong invoice is a P0 bug).

### 1.5 Success metrics
| Metric | Target |
| --- | --- |
| Invoices generated needing manual edit | < 5% |
| Time to complete a monthly rent cycle (owner effort) | < 30 min per 50 beds |
| Reminder → payment within 3 days | > 60% |
| Complaints closed within SLA | > 80% |
| Owner weekly active use during pilot | 5+ days/week |
| Pilot → paid conversion | > 50% |

---

## 2. Users and roles

| Role | Who | Access |
| --- | --- | --- |
| **Owner** | Business owner | Everything, including profit, subscription, user management, deletion |
| **Manager / warden** | On-site staff running daily ops | Rooms, tenants, invoices, payment recording, complaints, expenses, meter readings, notices. **No** profit view, no subscription, no user management |
| **Accountant** | Part-time bookkeeper | Read-only financials + expense entry + exports. No tenant ID proofs |
| **Staff** | Cook, cleaner, maintenance | Only complaints assigned to them + their own attendance. Optional in v1 (see OQ-4) |
| **NEXSTAY Admin** | Internal | Cross-organization support access, subscription management, audited impersonation |

Design rule: the Manager role must be genuinely usable, because owners delegate day-to-day work. Any screen an owner uses daily except profit/analytics must work for a Manager.

---

## 3. Core domain concepts

| Concept | Definition |
| --- | --- |
| **Organization** | The owner's business account. Holds properties, users, subscription |
| **Property** | One PG building |
| **Floor / Room / Bed** | Physical hierarchy. A room's sharing type determines its bed count. **The bed is the billable unit** |
| **Tenant** | A person, identified by phone number, independent of any stay |
| **Tenancy** | A tenant occupying a specific bed for a period, with agreed rent, deposit, billing day, notice period. All money hangs off the tenancy, not the tenant |
| **Invoice** | A bill for one tenancy for one billing period, made of line items |
| **Payment** | Money received against one or more invoices |
| **Meter reading** | Electricity units for a room (or bed) in a period, used to compute the electricity line item |
| **Expense** | Money the owner spent, by category |
| **Complaint** | A tenant-reported issue with a lifecycle and audit trail |

Non-negotiable modelling rules (rationale in SRS §5):
1. Occupancy is **derived** from active tenancies; bed status is a projection, never the source of truth.
2. An issued invoice is **immutable**; corrections happen by cancel-and-reissue or credit note.
3. All money is stored as **integer paise**.
4. Rent changes are **effective-dated**; past invoices never change retroactively.

---

## 4. Feature requirements

Priorities: **P0** must ship for pilot; **P1** must ship before charging money; **P2** later.

### 4.1 Onboarding and setup (P0)
- **PRD-1** Owner signs up with mobile number + OTP, sets name, business name, city.
- **PRD-2** Guided setup wizard: create property → define floors → add rooms in bulk (e.g. "floor 2, rooms 201–210, double sharing, ₹6,500/bed") → set billing defaults → add tenants.
- **PRD-3** Bulk room creation must be able to set up a 50-bed PG in under 10 minutes.
- **PRD-4** Tenant import from CSV or spreadsheet paste, with a preview and per-row validation errors, because every pilot owner already has a sheet.
- **PRD-5** Billing defaults per property: billing day (fixed day of month or per-tenancy anniversary), electricity mode, food mode, late-fee rule, notice period, currency/tax settings.
- **PRD-6** Setup checklist on the dashboard showing what's still missing before the first invoice run can succeed.

### 4.2 Property, rooms, beds (P0)
- **PRD-7** Multiple properties per organization, with a property switcher; all operational screens are property-scoped.
- **PRD-8** Room attributes: number/name, floor, sharing type, bed count, room type tags (AC/non-AC, attached bathroom, balcony, furnished), default rent per bed, amenities, notes.
- **PRD-9** Bed attributes: label (201-A), rent (defaults from room, overridable), status Vacant / Occupied / Notice Period / Blocked (with reason: maintenance, owner use).
- **PRD-10** Occupancy grid: colour-coded floor-by-floor view of every bed with tenant name and dues indicator; click a bed to see or start a tenancy.
- **PRD-11** Increasing sharing type adds beds without touching existing tenancies; decreasing below the occupied count is blocked with a message naming the blocking tenants.
- **PRD-12** Vacancy list with days-vacant per bed and estimated revenue lost, so owners act on it.
- **PRD-13** Rooms/beds can be archived, never hard-deleted while historical invoices reference them.

### 4.3 Tenants and tenancies (P0)
- **PRD-14** Tenant record: name, photo, phone (unique per organization), alternate phone, email, permanent address, occupation (student / working professional) and institution/company, gender, date of birth, blood group (optional), emergency contact (name, relation, phone — mandatory), ID proof documents with type and number.
- **PRD-15** ID proofs stored encrypted; numbers masked in the UI (last 4 visible); full view is Owner-only and audit-logged.
- **PRD-16** Tenancy creation: pick bed → tenant (new or existing) → move-in date → rent → security deposit (agreed, received, balance) → billing day → notice period → food included → optional recurring charges (laundry, parking) → optional agreement PDF upload.
- **PRD-17** Deposit tracking: agreed vs. received vs. refunded, with the refund happening only through move-out settlement.
- **PRD-18** One active tenancy per bed; one active tenancy per tenant per property.
- **PRD-19** Rent revision: new amount + effective date + reason; visible as a history on the tenancy.
- **PRD-20** Bed transfer: move a tenant to another bed mid-cycle, with pro-rated split across both beds on the invoice and full history of the transfer.
- **PRD-21** Notice: record notice date → system computes expected vacating date from notice period → bed shows Notice Period → appears in an "upcoming vacancies" list.
- **PRD-22** Move-out settlement: pro-rated final rent, final electricity, unpaid dues, damage/other deductions, deposit refund → single settlement statement (PDF) → tenancy closed, bed freed on the effective date.
- **PRD-23** Tenant profile page shows the full timeline: tenancies, invoices, payments, dues, complaints, notices, documents.
- **PRD-24** Search tenants by name, phone, room, bed across the organization.

### 4.4 Rent invoicing (P0) — the heart of the product
- **PRD-25** Automatic invoice generation on each tenancy's billing day, run by a scheduled job, for the coming period. Idempotent: one invoice per tenancy per period, ever.
- **PRD-26** Owner can preview the whole upcoming cycle as **drafts**, edit or add line items, then issue in bulk. **[A-1: drafts are the default; auto-issue is a per-property setting]**
- **PRD-27** Line item types: rent (pro-rated on partial months), electricity, food, recurring charge, one-off charge, late fee, discount/waiver, previous balance carry-forward.
- **PRD-28** Pro-ration rule: `rent × days_occupied / days_in_month`, rounded to the nearest rupee, rule stated on the invoice. **[A-2]**
- **PRD-29** Invoice numbering: per-property sequential, gapless, configurable prefix and financial-year reset.
- **PRD-30** Invoice states: Draft → Issued → Partially Paid → Paid, with Overdue as a derived flag, plus Cancelled (reason required) and Written Off (Owner only).
- **PRD-31** Issued invoices are immutable. A correction cancels and reissues, or adds a credit note; both link to the original.
- **PRD-32** PDF invoice with PG name/logo/address, tenant and bed details, period, itemised charges, electricity units and rate, previous balance, total, amount paid, balance due, due date, payment instructions (owner's UPI ID/QR), and the late-fee policy line.
- **PRD-33** Bulk actions: issue all, download all as a ZIP, send all via WhatsApp/SMS.
- **PRD-34** Late fee applied automatically per the property rule after the grace period, as a visible line item on the next invoice, and waivable by the Owner with a reason.
- **PRD-35** Ad-hoc invoice outside the rent cycle (damage recovery, one-time charge).

### 4.5 Payments and dues (P0)
- **PRD-36** Record payment: amount, date, method (cash, UPI, bank transfer, cheque, card), reference number, receiving user, note, optional receipt photo; allocate to one or more invoices oldest-first by default, overridable.
- **PRD-37** Partial payments and advance payments (credited to the tenancy and auto-applied to the next invoice).
- **PRD-38** Payment receipt PDF, shareable on WhatsApp.
- **PRD-39** Dues dashboard: total pending, ageing buckets (0–7 / 8–15 / 16–30 / 30+ days), per-tenant balance, sortable and filterable by room/floor; one-tap "remind" per row and "remind all".
- **PRD-40** Collection report per period: billed vs. collected vs. pending, collection %, by property.
- **PRD-41** Refunds and adjustments recorded explicitly, never as a negative payment.
- **PRD-42** Online payment collection is a **pluggable interface** in v1: the payments module exposes `PaymentProvider` with a `Manual` implementation, so a gateway can be added later without touching the ledger. Decision pending in OQ-1.

### 4.6 Electricity billing (P0)
- **PRD-43** Three modes per property, overridable per room: included in rent, flat monthly amount, per-unit metered.
- **PRD-44** Metered mode: record previous and current reading per meter (room-level or bed-level), unit rate, plus optional fixed charge; the system computes units and amount and splits it equally across occupied beds in the room for the days they were occupied. **[A-3]**
- **PRD-45** Reading entry screen designed for fast bulk entry by staff on a phone, showing last month's reading, flagging implausible jumps (>3× the 3-month average) before saving.
- **PRD-46** Optional photo of the meter attached to a reading, for dispute resolution.
- **PRD-47** Missing readings block the electricity line item, not the whole invoice; the owner is warned in the cycle preview.

### 4.7 Expenses (P0)
- **PRD-48** Expense entry: property, category (electricity bill, food/groceries, staff salary, maintenance, repairs, internet, water, gas, rent/lease, taxes, misc), amount, date, paid-to, payment method, note, receipt image.
- **PRD-49** Recurring expense templates with a reminder on the due day.
- **PRD-50** Expense list with filters and monthly summary by category, plus month-over-month comparison.
- **PRD-51** Staff salary payments recorded in the employee module post automatically to the staff-salary expense category — entered once, not twice.

### 4.8 Dashboard and reports (P0)
- **PRD-52** Owner dashboard: this month's billed revenue, collected, pending, expenses, net profit, occupancy % and vacant beds, open complaints, upcoming vacancies, setup/action items.
- **PRD-53** 12-month trend charts: revenue, collection rate, occupancy, expenses, profit.
- **PRD-54** Per-property comparison for multi-property owners (P1).
- **PRD-55** Reports: rent roll (per bed: tenant, rent, paid, due), collection report, dues ageing, expense report, occupancy report, tenant register (for police/regulatory requests), P&L per property per month.
- **PRD-56** CSV/Excel export for every report; PDF for P&L and rent roll (P1).
- **PRD-57** Every dashboard number must reconcile to the underlying records for the same period — a "view underlying records" drill-down is required, not optional.

### 4.9 Complaints (P0)
- **PRD-58** Complaint created by owner/staff on a tenant's behalf (tenant has no login in this release), or via a **public tokenised link** the tenant can open from WhatsApp with no signup. **[A-4]**
- **PRD-59** Fields: property, room/bed, tenant, category (electrical, plumbing, water, cleaning, food, wifi, furniture, appliance, security, other), description, priority (low/medium/high/urgent), photos/video.
- **PRD-60** Lifecycle: Pending → In Progress → Resolved, plus Rejected (reason required) and Reopened. Assignment to an employee, due-by based on priority SLA, internal notes vs. tenant-visible updates, full event timeline.
- **PRD-61** Tenant notified via WhatsApp/SMS on assignment and resolution, with a link to view status.
- **PRD-62** Complaint reports: open by category/age, SLA breaches, median resolution time, repeat issues per room (P1).

### 4.10 Employees (P1)
- **PRD-63** Employee record: name, photo, role, phone, ID proof, joining date, monthly salary, salary cycle, emergency contact, address, documents.
- **PRD-64** Attendance: daily mark (present / absent / half-day / paid leave / unpaid leave / week-off), bulk mark for a day, monthly grid view.
- **PRD-65** Salary computation from attendance rules (deduction for unpaid leave), advances, and salary payments recorded → posts to expenses.
- **PRD-66** Assignment of complaints to employees and a simple work log per employee.
- **PRD-67** Optional staff login with a heavily restricted scope (OQ-4).

### 4.11 Notices and messaging to tenants (P0)
- **PRD-68** Compose a notice: title, message, optional attachment, audience = all tenants in a property / specific floors / specific rooms / selected tenants / all tenants with dues.
- **PRD-69** Channels: WhatsApp (templated) with SMS fallback; the notice is also stored in the tenant's timeline.
- **PRD-70** Notice types: rent reminder, maintenance, emergency, general announcement — driving template choice and urgency.
- **PRD-71** Automated rent reminders on a configurable schedule (e.g. 3 days before due, on due day, +3, +7, +15 days), pausable per tenant, never sent to a fully paid tenancy.
- **PRD-72** Delivery log per message (sent / delivered / failed / read where the channel reports it), with a retry action.
- **PRD-73** Message templates are pre-approved and editable in a limited way (variables only), because WhatsApp Business templates require provider approval.

### 4.12 Subscription and account management (P1)
- **PRD-74** Plans metered on managed beds: Starter ≤50 beds ₹4,999/mo, Growth ≤100 beds ₹8,999/mo, Enterprise 100+ custom. 14-day free trial. **[A-5: bed count = beds created across all properties]** (OQ-2)
- **PRD-75** Plan limit behaviour: at 90% show a warning; over the limit, block creating new beds and start a 7-day grace period before restricting invoice generation. Never block access to existing data or block payment recording.
- **PRD-76** Subscription states: trialing, active, past_due, cancelled, with dunning notices.
- **PRD-77** Organization users: invite by phone with a role, deactivate, view an access log.
- **PRD-78** Data export of everything the organization owns, on demand (also a trust and DPDP requirement).

### 4.13 Platform behaviours (P0)
- **PRD-79** Mobile-responsive throughout; owners will use this on a phone more than a laptop. Occupancy grid, dues list, payment recording, complaint update and meter entry must be fully usable one-handed on a 5-inch screen.
- **PRD-80** Offline tolerance: meter reading and payment recording forms retain input and retry on reconnect. **[A-6: local draft persistence, not full offline sync]**
- **PRD-81** Every destructive action is confirmable and reversible or audit-logged.
- **PRD-82** All actions on money, tenancies, complaints and permissions are audit-logged with actor, before/after and timestamp.
- **PRD-83** English UI in v1, with all strings externalised for Hindi next.

---

## 5. Key flows (acceptance-level)

### 5.1 Monthly rent cycle
1. Two days before the billing day, staff enter meter readings (PRD-45).
2. On the billing day at 06:00 IST the job creates draft invoices for every active tenancy (PRD-25).
3. Owner opens "Rent cycle — Nov 2026": totals, per-tenancy rows, warnings (missing reading, unusual amount, tenant on notice).
4. Owner edits what's needed and clicks **Issue all** → invoices become Issued, PDFs generated, WhatsApp messages sent.
5. Payments arrive; owner or manager records each one (PRD-36) or marks paid from the dues list.
6. Reminders escalate automatically (PRD-71) until the balance is zero.
7. End of month: collection report and P&L (PRD-55).

**Acceptance:** a 60-bed PG completes steps 3–4 in under 10 minutes; re-running the job creates no duplicates; a tenant who moved in on the 12th is billed pro-rata for that month only.

### 5.2 New tenant move-in
Vacant bed → New tenancy → tenant details + ID proof + emergency contact → rent, deposit, billing day → deposit received recorded → welcome WhatsApp with PG rules and payment instructions → first (pro-rated) invoice issued immediately, not waiting for the next cycle.

**Acceptance:** bed becomes Occupied on the move-in date; occupancy % and dues update immediately; first invoice pro-rated correctly.

### 5.3 Move-out
Notice recorded → bed shows Notice Period and appears in upcoming vacancies → on vacating: final electricity reading, deductions entered → settlement statement shows final rent (pro-rated), dues, deductions, deposit refund, net payable either way → refund recorded → tenancy closed, bed Vacant, tenant history retained.

**Acceptance:** deposit is never silently absorbed — the settlement always shows it explicitly; a tenant with dues exceeding the deposit produces a net-receivable, not a negative refund.

### 5.4 Complaint
Tenant sends a WhatsApp message or uses the tokenised link → complaint logged with photos → auto-assigned or manually assigned with SLA → status updates notify the tenant → resolution note closes it → tenant can reopen within 48h via the link.

---

## 6. Out-of-scope guardrails (avoid scope creep)
Do **not** build in this release: tenant login, gateway payments, reels, marketplace listings, AI suggestions, chat/messaging inbox, biometric attendance, accounting integrations (Tally/Zoho), GST invoicing beyond a configurable tax line, food menu management, visitor/gate management, laundry as a separate module, or a native mobile app.

Each of these has a clean insertion point in the architecture (see SRS §3) so deferring them costs nothing structurally.

---

## 7. Assumptions
- **A-1** Invoices are generated as drafts and issued by the owner; auto-issue is opt-in per property.
- **A-2** Pro-ration is calendar-day based on the actual days in the month.
- **A-3** Metered electricity is split equally across occupied beds in a room, weighted by days occupied.
- **A-4** Tenants interact via WhatsApp and tokenised links only in this release — no tenant login.
- **A-5** Subscription bed count = beds created, not beds occupied.
- **A-6** "Offline tolerance" means local draft persistence and retry, not a full offline-first sync engine.
- **A-7** One organization = one owner's business; franchise/aggregator hierarchies are out of scope.
- **A-8** Invoices need a configurable tax line but full GST compliance/filing is out of scope for v1.

## 8. Open questions
- **OQ-1** Rent collection: keep it record-only in v1 (owner's own UPI, tenant pays outside the platform), or integrate a gateway now? Record-only is assumed; it is far faster to ship and avoids settlement/escrow regulation, but loses auto-reconciliation.
- **OQ-2** Subscription bed count on created vs. occupied beds.
- **OQ-3** Default late-fee policy, and is it owner-configurable per property?
- **OQ-4** Do staff (cook/cleaner/maintenance) get logins in v1?
- **OQ-5** Do owners need GST-compliant invoices at launch (i.e. are pilot PGs GST-registered)?
- **OQ-6** WhatsApp Business API provider — do you have an account, or should reminders start as SMS/manual share links?
- **OQ-7** Are there existing MVP wireframes to match for the ERP screens?
- **OQ-8** Do pilot PGs charge per-bed or per-room electricity, and are meters per room or per floor? Per-floor meters would need a different split rule.
- **OQ-9** Food: is it always included in rent for pilot PGs, or billed separately with attendance-based deductions?
