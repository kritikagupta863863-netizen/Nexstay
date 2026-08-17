# NEXSTAY — Product Requirements Document

Status: Draft v1
Owner: Shikhar Agnihotri (founder)
Sources: `docs/source-documents/NEXSTAY_Overview.pdf`, `docs/source-documents/NEXSTAY_Business_Plan.pdf`
Related: [ROADMAP.md](ROADMAP.md)

Open questions are marked **[OQ-n]** and collected in section 12. Where a decision was needed to write a testable requirement, an assumption is stated as **[A-n]** (section 11) rather than left blank.

---

## 1. Overview

### 1.1 Problem
India's PG industry serves 20M+ students and working professionals, but most properties are run on WhatsApp, Excel, paper receipts and phone calls. Owners lose rent, mismanage occupancy, lose tenant records and have no business insight. Tenants face fragmented listings, hidden charges and no visibility into amenities or policies.

### 1.2 Product
NEXSTAY is one platform with three surfaces on a shared backend:

1. **Owner ERP** — run the PG business (rooms, tenants, rent, expenses, complaints, staff, invoices, analytics).
2. **Tenant App** — live in the PG (rent, invoices, complaints, notices, agreements).
3. **Marketplace** — find the PG (search, reels, transparent pricing, AI matching, enquiries).

### 1.3 Goals
- G1: A PG owner can fully replace Excel/WhatsApp/paper for rent, occupancy, expenses and complaints.
- G2: Rent collection cycle time and pending-rent leakage drop measurably vs. the owner's manual baseline.
- G3: A tenant can see the complete cost of a PG before enquiring — no hidden charges.
- G4: Marketplace generates qualified inbound enquiries that owners can convert inside the ERP.
- G5: The platform bills owners on a per-bed SaaS plan.

### 1.4 Non-goals (v1)
Rental agreements with legal e-sign, smart locks/access control, embedded finance and tenant credit scoring, hostel/hotel/PMS use cases, roommate social network, in-app chat between strangers.

### 1.5 Success metrics
| Metric | Target |
| --- | --- |
| Pilot PGs live end-to-end | 3 in first month post-Phase 1 |
| Rent invoices generated automatically vs. manually corrected | >95% need no manual edit |
| Tenant app adoption per PG | >70% of active tenants activated |
| Complaint median resolution time | Tracked from day 1, then −30% |
| Paid conversion after pilot | >50% of pilot PGs on a paid plan |
| Marketplace enquiry → tenancy | >10% |

---

## 2. Personas

| Persona | Context | Needs |
| --- | --- | --- |
| **Owner** (Rajesh, 2 PGs, 60 beds) | Not very technical, lives on WhatsApp, checks phone not desktop | Know who hasn't paid, send reminders in one tap, see profit, fill vacancies |
| **Manager / warden** (staff) | On-site daily, handles complaints and meter readings | Limited-scope access: no revenue/profit, yes complaints, attendance, readings |
| **Tenant** (Priya, student) | Mobile-only, wants no arguments about bills | Clear dues, downloadable invoices, complaint that doesn't get ignored |
| **Prospective tenant** (Aman, new to the city) | Browsing on phone, distrusts listings | Real photos/video, true total cost, roommate fit, easy enquiry |
| **NEXSTAY admin** (internal) | Onboarding and support | Verify PGs, manage subscriptions, impersonate for support, moderate media |

---

## 3. Key user journeys

**J1 Owner onboarding** — signup (mobile OTP) → create PG (address, floors, rooms, beds, sharing types, rent) → add existing tenants with rent start dates and deposits → set electricity rule and billing day → invite tenants to app → first invoice cycle runs.

**J2 Monthly rent cycle** — on billing day the system generates invoices per active tenancy (rent + electricity + recurring/one-off charges) → tenants notified (push + WhatsApp/SMS) → tenant pays via UPI or owner records offline payment → invoice moves Unpaid → Partially Paid → Paid → dues dashboard and escalating reminders for overdue.

**J3 Complaint** — tenant raises complaint with category, description, photos/video → appears in owner/manager inbox → assigned to staff, status Pending → In Progress → Resolved with resolution note → tenant notified and can reopen within 48h **[A-1]**.

**J4 Vacancy to move-in** — owner marks bed vacant and publishes/refreshes listing (photos, reels, transparent pricing) → prospective tenant discovers via search or reels → views detail page with full cost breakdown → sends enquiry / booking request → owner sees it in ERP inbox, responds, converts to tenancy → bed occupancy and subscription bed count update.

**J5 Move-out** — owner initiates move-out with notice date → system computes pro-rata rent, final electricity, deductions, deposit refund → final settlement invoice → tenancy closed, bed freed, tenant app switches to read-only history.

---

## 4. Functional requirements — Owner ERP

Priority: **P0** = required for pilot, **P1** = required for paid launch, **P2** = later.

### 4.1 Property, rooms and beds (P0)
- FR-1 Owner can create multiple properties under one account; each has name, address, geo coordinates, contact, photos.
- FR-2 Property has floors; floors have rooms; rooms have a sharing type (single/double/triple/…) and a bed count that determines the number of bed records.
- FR-3 Each bed has: rent amount, status (Vacant / Occupied / Blocked / Notice Period), and optional per-bed rent override.
- FR-4 Room supports category/type labels (AC/non-AC, attached bath, balcony) and amenities.
- FR-5 Occupancy view: property-wide grid showing every bed's status and current tenant; filterable by floor/status/sharing type.
- FR-6 Vacancy report: count and list of vacant beds, days-vacant per bed.
- AC: changing a room's sharing type from 2 to 3 adds a bed without disturbing existing tenancies; reducing below occupied count is blocked with a clear error.

### 4.2 Tenants and tenancies (P0)
- FR-7 Tenant record: name, photo, phone (unique login identity), email, permanent address, occupation (student/professional), emergency contact (name, relation, phone), ID proof uploads (Aadhaar/PAN/college ID) **[A-2: stored encrypted, masked in UI, visible only to owner role]**.
- FR-8 Tenancy links tenant ↔ bed with: move-in date, agreed rent, security deposit, billing day, notice period days, food included flag, expected move-out date.
- FR-9 A bed can have only one active tenancy at a time; a tenant only one active tenancy **[OQ-1: allow a tenant to hold beds in two PGs?]**.
- FR-10 Tenant history: all past tenancies, invoices, payments, complaints, notices; retained after move-out.
- FR-11 Rent change on an active tenancy is recorded with an effective date; past invoices are never retroactively altered.
- FR-12 Move-out flow per J5, producing a final settlement with pro-rata rent, dues, deductions and deposit refund.
- AC: assigning a tenant to an occupied bed is rejected; move-out frees the bed on the effective date, not immediately.

### 4.3 Rent, invoicing and billing (P0)
- FR-13 Scheduled job generates invoices for each active tenancy on its billing day for the upcoming period; idempotent (never two invoices for the same tenancy+period).
- FR-14 Invoice line items: rent (pro-rated for partial months), electricity, food charge, one-off charges (damage, late fee), recurring charges (laundry), discounts.
- FR-15 Electricity: per-room or per-bed meter readings (previous, current, units, rate) → amount split across the room's occupied beds; also supports flat monthly charge and "included in rent" **[A-3]**.
- FR-16 Invoice states: Draft → Issued → Partially Paid → Paid → Overdue → Cancelled; Cancelled requires a reason and keeps an audit record.
- FR-17 Payment recording: online (UPI/Razorpay, auto-reconciled via webhook) and offline (cash/bank/UPI-direct with reference note, recorded by owner/manager).
- FR-18 Downloadable PDF invoices and payment receipts, with PG branding and full charge breakdown.
- FR-19 Dues dashboard: pending rent by tenant/room, ageing buckets (0–7, 8–15, 16–30, 30+ days).
- FR-20 Automated reminders: before due date, on due date, and configurable escalations after; via push + WhatsApp/SMS.
- FR-21 Late fee rule (fixed or %, after N days grace), applied automatically and visible on the invoice **[OQ-2: default policy?]**.
- FR-22 All money stored as integer paise; every state change is an append-only ledger entry with actor and timestamp.
- AC: a mid-month move-in produces a pro-rated first invoice; a partial payment leaves an exact remaining balance; regenerating a cycle twice does not duplicate invoices.

### 4.4 Expenses (P0)
- FR-23 Expense record: property, category (electricity, food, maintenance, staff salary, internet, repairs, misc), amount, date, vendor, note, receipt image; recurring expense templates.
- FR-24 Expense summary per month with category breakdown and month-over-month comparison.

### 4.5 Analytics and dashboard (P0)
- FR-25 Owner dashboard: monthly revenue (billed), collected, pending, expenses, net profit, occupancy %, new move-ins/move-outs, open complaints.
- FR-26 Trends over the last 12 months for revenue, collection rate, occupancy and expenses.
- FR-27 Multi-property consolidated view plus per-property drill-down (P1).
- FR-28 Exports: invoices, payments, expenses, tenants as CSV (P1).
- AC: dashboard numbers reconcile exactly with invoice/payment/expense records for the same period.

### 4.6 Complaints (P0)
- FR-29 Complaint fields: category (electrical, plumbing, cleaning, food, wifi, furniture, other), description, media (images/video), raised-by, room/bed, priority.
- FR-30 Statuses Pending / In Progress / Resolved (+ Rejected with reason); assignment to an employee; internal notes vs. tenant-visible updates; timeline of events.
- FR-31 Tenant notified on every status change; resolution requires a note **[A-1: reopen within 48h]**.
- FR-32 Complaint analytics: open count, by category, median resolution time (P1).

### 4.7 Employees (P1)
- FR-33 Employee profile: name, photo, role (manager, cook, cleaner, security, maintenance), phone, salary, joining date, emergency contact, ID proof.
- FR-34 Attendance: daily present/absent/half-day/leave, monthly summary.
- FR-35 Salary tracking: monthly salary due, payments recorded, advances; feeds the staff-salary expense category automatically.
- FR-36 Staff app/portal access with restricted scope (complaints assigned to them, attendance, meter readings) — no financials **[OQ-3: staff login in v1 or owner-entered only?]**.

### 4.8 Notifications and announcements (P0)
- FR-37 Owner composes a notice with title, body, optional attachment, and audience: all tenants / specific property / specific rooms / selected tenants.
- FR-38 Channels: in-app + push always; WhatsApp/SMS for rent reminders and emergencies **[A-4: WhatsApp via BSP template messages]**.
- FR-39 Notice types (rent reminder, maintenance, emergency, general) drive priority and styling; emergency bypasses quiet hours.
- FR-40 Delivery log: who received/read what.

### 4.9 PG profile and listing (P1)
- FR-41 PG profile: description, images, room photos, video, amenities, food menu/details, pricing per sharing type, security deposit, rules and policies, notice period, nearby landmarks, availability status.
- FR-42 Listing publish/unpublish; publishing requires the mandatory transparency fields (section 5.4) to be complete.
- FR-43 Promotions: banners, offer text with validity, featured-listing slot (paid) (P2).
- FR-44 Admin verification badge on a PG after internal review (P1).

### 4.10 Subscription and account (P1)
- FR-45 Plans metered by managed bed count: Starter ≤50 beds ₹4,999/mo, Growth ≤100 beds ₹8,999/mo, Enterprise 100+ custom.
- FR-46 Bed count computed from beds created (not occupied) **[OQ-4: confirm — beds created vs. occupied changes revenue and gaming risk]**; exceeding the plan limit prompts an upgrade and blocks new bed creation after a grace period.
- FR-47 Subscription lifecycle: trial, active, past due, cancelled; invoices for NEXSTAY's own SaaS fees; onboarding fee as a one-off charge.
- FR-48 Roles and permissions: Owner (all), Manager (operations, no profit/subscription), Staff (assigned tasks), Accountant (read financials) **[A-5]**.

---

## 5. Functional requirements — Marketplace

### 5.1 Search and explore (P1)
- FR-49 Filter by city/locality, budget range, sharing type, room type, gender preference, food availability, amenities, availability date.
- FR-50 Text search over locality, landmark and PG name; geo search "near me" and near a chosen landmark (college/office) with radius and distance shown.
- FR-51 Result card shows: cover media, PG name, locality, distance, starting rent, sharing types available, food included, verified badge, rating (P2).
- FR-52 Sort by relevance, price, distance, newest.
- FR-53 SEO-indexable city and locality landing pages with server-rendered listings.

### 5.2 Reels / video feed (P2)
- FR-54 Vertical short-video feed of room tours; autoplay, mute toggle, swipe navigation.
- FR-55 Each reel links to its PG with inline CTAs: Enquire, Call, Save, Share.
- FR-56 Owner uploads video from the ERP; transcoded to streaming renditions; moderation queue before going live.
- FR-57 Feed ranking blends recency, engagement, distance to the viewer and completeness of the listing.

### 5.3 PG detail page (P1)
- FR-58 Gallery (images + video), pricing per sharing type, amenities, food details, room types with availability, rules and policies, nearby places, map, owner response time.
- FR-59 CTAs: Enquire (form), Request booking (dates + sharing preference), Call/WhatsApp **[OQ-5: expose the owner's raw phone number, or mask it?]**.

### 5.4 Transparency system (P1)
- FR-60 Mandatory cost breakdown before publish: base rent per sharing type, security deposit, electricity basis (included / per-unit rate / flat), food charges (included or amount), maintenance, one-time onboarding/registration charges, notice period, lock-in period, refund policy.
- FR-61 The detail page shows a computed "monthly total" and "move-in total" (first rent + deposit + one-time charges).
- FR-62 Any charge not declared in the listing may not be applied to a NEXSTAY-managed tenancy without the tenant accepting it in-app **[A-6 — this is the core trust promise; enforce it in the invoicing layer]**.

### 5.5 AI matching and roommate compatibility (P2)
- FR-63 Tenant preference profile: budget, locality/commute anchor, sharing type, food (veg/non-veg/either), smoking, drinking, sleep schedule, cleanliness, student vs. professional, gender preference.
- FR-64 Ranked recommendations with a human-readable reason ("within ₹500 of your budget, 1.2 km from your college, veg-only kitchen").
- FR-65 Compatibility score between a prospective tenant and a specific room's current occupants, computed only from preferences occupants consented to share; never exposes another tenant's identity or protected attributes.
- FR-66 Recommendations must be explainable and must not rank on paid promotion without labelling it "Promoted".

### 5.6 Enquiries (P1)
- FR-67 Enquiry captures name, phone (OTP-verified), move-in date, budget, sharing preference, message; rate-limited to prevent spam.
- FR-68 Owner inbox with status (New / Contacted / Visit scheduled / Converted / Lost), reply, and one-click convert-to-tenancy that pre-fills the tenant record.
- FR-69 Owner response-time metric shown on the listing.

---

## 6. Functional requirements — Tenant App

- FR-70 Home: current dues with due date and pay button, this month's invoice summary, open complaints, latest notices, PG contact.
- FR-71 Invoices list with full line-item breakdown, status, payment history, PDF download; electricity shows units and rate, not just an amount.
- FR-72 Pay rent via UPI; payment status reflected immediately; receipt issued **[OQ-6: does money settle to the owner directly or route through NEXSTAY?]**.
- FR-73 Raise a complaint with category, description, up to 5 images or 1 video; track timeline; rate the resolution (P2).
- FR-74 Notices feed with read state; push notifications for dues, complaint updates, emergencies.
- FR-75 Profile: photo, contacts, emergency contact, ID proof upload, room/bed details, stay duration, preferences (feeds compatibility with consent).
- FR-76 Documents: rent agreement PDF (uploaded by owner in v1), house rules, deposit record.
- FR-77 Move-out request with intended date; shows notice-period implications and expected settlement estimate.
- FR-78 Read-only access to history after move-out.
- AC: a tenant can never see another tenant's financial or identity data; a tenant sees the same invoice numbers the owner sees.

---

## 7. Cross-cutting requirements

### 7.1 Authentication and identity (P0)
- FR-79 Mobile OTP as the primary login for all roles; email+password optional for owners; separate owner and tenant onboarding flows.
- FR-80 One phone number can hold multiple roles (an owner may also be a tenant elsewhere); role selection at login when ambiguous.
- FR-81 Session management, device logout, and OTP rate limiting/lockout.
- FR-82 Tenant accounts are invited by the owner or self-signup and then linked to a tenancy by the owner.

### 7.2 Media (P0)
- FR-83 Image upload with client-side compression, server-side validation and virus/type checks; video upload with transcoding for reels and complaint clips; signed, expiring URLs for private media (ID proofs, complaint media).

### 7.3 Notifications infrastructure (P0)
- FR-84 Push (FCM), WhatsApp and SMS via a provider abstraction with retries, per-channel opt-out (except transactional/legal), and delivery logging.

### 7.4 Audit and data integrity (P0)
- FR-85 Append-only audit log for money, tenancy and complaint state changes: actor, before/after, timestamp, IP.
- FR-86 Soft delete for tenants, tenancies, invoices; hard delete only via admin with reason.

### 7.5 Non-functional
- NFR-1 Mobile-first responsive web; tenant app usable on low-end Android and 3G.
- NFR-2 p95 API latency < 500 ms for reads, < 1.5 s for writes; dashboard loads < 2.5 s on 3G.
- NFR-3 Availability target 99.5% in year one; nightly automated backups with a tested restore.
- NFR-4 Data residency in India; encryption in transit and at rest; ID proofs and payment data encrypted at the field level.
- NFR-5 Compliance posture for India's DPDP Act: consent for personal data, purpose limitation, deletion request handling.
- NFR-6 Localisation-ready (English v1, Hindi next); currency ₹, dates dd/mm/yyyy, Asia/Kolkata.
- NFR-7 Accessibility: WCAG 2.1 AA for the marketplace and tenant app basics.
- NFR-8 Observability: structured logs, error tracking, alerts on failed invoice jobs and payment webhook failures.

---

## 8. Data model (logical)

```
Organization ─┬─ User (owner/manager/staff/accountant, role-scoped)
              └─ Property ─┬─ Floor ─ Room ─ Bed
                           ├─ Listing (media[], pricing_breakdown, amenities, policies, published)
                           ├─ Expense, Employee ─ Attendance, SalaryPayment
                           └─ Enquiry / BookingRequest

Tenant ─ Tenancy (tenant, bed, move_in, move_out, rent, deposit, billing_day, notice_days)
Tenancy ─┬─ Invoice ─ InvoiceLineItem
         │            └─ Payment (method, txn_ref, gateway_payload)
         ├─ MeterReading (room|bed, period, prev, curr, rate)
         └─ Complaint ─ ComplaintEvent, Media[]

Notification / Notice ─ NotificationRecipient (delivery, read_at)
Subscription (organization, plan, bed_count, period, status) ─ SaasInvoice
AuditLog (actor, entity, action, before, after, at)
TenantPreference (consent flags) ─ used by matching / compatibility
```
Rules: money in integer paise; invoices immutable once Issued (corrections via credit note or cancel+reissue); occupancy derived from Tenancy, never stored only on Bed.

---

## 9. Release scope

| Release | Contents | Definition of done |
| --- | --- | --- |
| **M1 Pilot ERP** | 4.1–4.6, 4.8, 7.1–7.4 | 3 real PGs run a full rent cycle, complaints and expenses without Excel |
| **M2 Tenant App** | Section 6, push, UPI pay | >70% tenant activation in pilot PGs; tenants pay in-app |
| **M3 Marketplace v1** | 4.9, 5.1, 5.3, 5.4, 5.6 | Listings live and SEO-indexed; enquiries convert inside the ERP |
| **M4 Commercialise** | 4.7, 4.10, 4.5 multi-property, exports | Owners on paid plans with metered beds and enforced limits |
| **M5 Differentiate** | 5.2 reels, 5.5 AI matching, AI profit suggestions, promotions | Reels feed live with moderation; AI suggestions grounded in real analytics |

AI profit suggestions (from the Overview doc) sit in M5 deliberately: they require several months of real revenue, occupancy and expense data to be credible, and must cite the underlying numbers rather than free-form generate advice.

---

## 10. Analytics and instrumentation
Track: owner activation funnel (signup → property created → tenants added → first invoice → first payment), invoice generation success/failure, reminder → payment conversion by channel, complaint lifecycle timings, marketplace search → detail → enquiry → conversion, reel watch-through, tenant app DAU/MAU per PG, subscription MRR and churn.

## 11. Assumptions
- **A-1** Complaints can be reopened by the tenant within 48h of resolution.
- **A-2** ID proofs are stored encrypted, masked in the UI, and visible only to the Owner role.
- **A-3** Electricity supports three modes: included in rent, flat monthly amount, per-unit metered split across occupied beds in the room.
- **A-4** WhatsApp messaging goes through a Business Solution Provider using approved templates; SMS is the fallback.
- **A-5** Roles in v1: Owner, Manager, Staff, Accountant, plus internal Admin.
- **A-6** Undeclared charges cannot be invoiced without explicit in-app tenant acceptance.
- **A-7** English-only UI in v1; Hindi in a later release.
- **A-8** One owner organization may hold multiple properties from day one, but consolidated multi-property analytics ships in M4.

## 12. Open questions
- **OQ-1** Can a tenant hold active tenancies in two PGs simultaneously?
- **OQ-2** Default late-fee policy (grace days, fixed vs. percentage) — and is it owner-configurable per PG?
- **OQ-3** Do staff get their own logins in v1, or does the owner/manager enter everything?
- **OQ-4** Is the subscription bed count based on beds created or beds occupied?
- **OQ-5** Should owner phone numbers be masked on the marketplace (call proxy) to protect the enquiry funnel?
- **OQ-6** Payment flow: settle directly to the owner's account (NEXSTAY records only) or route/escrow through NEXSTAY? This is the single biggest architectural and regulatory decision.
- **OQ-7** Deposit handling: does NEXSTAY ever hold deposits, or only record them?
- **OQ-8** Who verifies a PG for the "verified" badge, and what is the checklist?
- **OQ-9** Tenant-facing ratings/reviews of PGs — in scope, and how is retaliation against a resident tenant prevented?
- **OQ-10** Existing wireframes/designs from the business plan ("MVP wireframes completed") — can they be shared so the UI matches them?
