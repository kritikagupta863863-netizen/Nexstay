# NEXSTAY — Requirements Summary & Build Roadmap

Source: `NEXSTAY_Overview.pdf`, `NEXSTAY_Business_Plan.pdf`.
Note: `Nexstay_Pitch_Deck_-_BeyondX.pdf` could not be read — its PDF content streams are corrupted (zlib "incorrect header check" in both poppler and MuPDF). Please re-export/re-share it if it contains requirements not in the other two docs.

---

## 1. What we're building

Three products on one backend:

| Product | Primary user | Purpose |
|---|---|---|
| PG Management ERP (web dashboard) | PG owner + staff | Rooms, tenants, rent, expenses, complaints, employees, invoices, analytics |
| PG Marketplace (web + mobile) | Prospective tenant | Discovery, reels feed, transparent pricing, AI matching, booking enquiry |
| Tenant App (mobile-first) | Resident tenant | Rent, invoices, complaints, agreements, notices |

Revenue: SaaS subscription per bed count (Starter ≤50 beds ₹4,999/mo, Growth ≤100 beds ₹8,999/mo, Enterprise custom) + onboarding fees + marketplace promotions + premium AI. This means **billing/subscription + bed-count metering are product requirements, not just business slides.**

## 2. Functional requirements (extracted, grouped)

### Owner ERP
- **Rooms**: floors, room categories, sharing type, bed-level occupancy, room-wise rent, vacancy tracking.
- **Tenants**: profile, emergency contact, ID proof upload, room/bed assignment, stay history, per-tenant rent ledger.
- **Billing & invoices**: auto monthly rent invoice generation, electricity bills (meter reading → amount), additional charges, receipts, PDF download, payment tracking, due reminders.
- **Revenue & profit**: monthly revenue, collected vs pending, expense breakdown, occupancy analytics, profit overview.
- **Expenses**: electricity, food, maintenance, staff salary, internet, repairs, misc.
- **Employees**: profile, role, salary, attendance, emergency contact, work tracking.
- **Complaints**: inbox from tenants, status (Pending / In Progress / Resolved), image & video attachments, assignment to staff, resolution notes.
- **Notifications**: broadcast to all tenants / by room / selected tenants; rent reminders, maintenance, emergency, announcements.
- **PG profile**: images, room photos, amenities, food details, pricing, deposit, rules, availability → feeds the marketplace listing.
- **Marketplace promotions**: banners, images, videos, offers, featured listing.
- **Multi-property**: one owner account managing several PGs (from business plan).

### Marketplace
- Explore with filters: location, budget, facilities, sharing type, room type, availability; smart search; nearby discovery (geo).
- Reels/short-video feed of room tours, with contact / enquiry / booking-request CTAs.
- PG detail page: media, pricing, facilities, room types, nearby places, food, rules, deposit, contact.
- **Smart transparency**: explicit breakdown of rent, electricity basis, security deposit, food charges, extra fees, notice period, room condition.
- AI matching on preference/budget/lifestyle/location/facilities.
- Roommate compatibility preferences (veg/non-veg, smoking, student/professional, lifestyle).

### Tenant app
- Dashboard: rent status, invoices, electricity bills, agreements, PG details, complaint tracker, notifications.
- Profile with photo, contacts, emergency contact, ID proof, room, stay duration, preferences.
- Raise complaint with media + description, track progress.
- Download/store invoices; payment confirmations; due reminders.

### Cross-cutting
- Auth: separate onboarding for owners vs tenants; mobile OTP + email; profile setup; role-based access (owner / manager / staff / tenant).
- Media storage (images, video for reels and complaints) with transcoding for video.
- Payments: UPI-first rent collection + subscription billing.
- Audit trail on money and complaint events.

## 3. Recommended architecture (Indian PG SMB context, small team)

- **Frontend web**: Next.js (App Router) + TypeScript + Tailwind + shadcn/ui. One repo serves owner dashboard + marketplace (public, SEO matters for PG discovery — SSR is a real advantage).
- **Mobile**: React Native (Expo) for the tenant app + marketplace reels, sharing the TypeScript API client. Start with a mobile-web PWA if you must ship faster, but reels/notifications want native.
- **Backend**: NestJS (TypeScript, same language as web) or FastAPI if you prefer Python for the AI parts. Modular monolith, not microservices, at this stage.
- **DB**: PostgreSQL (money/ledger integrity, relational occupancy model) + Redis for queues/cache. `pgvector` later for AI matching embeddings.
- **Jobs**: BullMQ/Celery for invoice generation, reminders, video transcoding, notification fan-out.
- **Storage/CDN**: S3-compatible (Cloudflare R2 or AWS S3 + CloudFront). Video via Cloudflare Stream or Mux to avoid building transcoding.
- **Payments**: Razorpay (UPI collect, payment links, subscriptions for your own SaaS billing, webhooks for reconciliation).
- **Comms**: MSG91/Gupshup for WhatsApp + SMS OTP and rent reminders (WhatsApp matters — it's the channel owners already use); FCM for push.
- **AI**: LLM API (OpenAI/Gemini) behind your own service layer for profit suggestions and matching explanations; deterministic analytics SQL first, LLM only for narrative/recommendation.
- **Infra**: single managed Postgres + containers on Railway/Render/AWS ECS; Sentry + structured logs; staging + prod.

### Core data model (first cut)
`Organization(owner) → Property(PG) → Floor → Room → Bed`
`Tenant`, `Tenancy(tenant, bed, start, end, rent, deposit, notice_period)`
`Invoice(tenancy, period, line_items[])`, `Payment(invoice, method, txn_ref)`, `MeterReading(room|bed, period, units)`
`Expense(property, category, amount, date)`, `Employee`, `Attendance`, `SalaryPayment`
`Complaint(tenancy, category, status, media[], events[])`, `Notification`, `Listing(property, media[], pricing_breakdown, amenities)`, `Enquiry/BookingRequest`
`Subscription(organization, plan, bed_count, period)`
Keep money in integer paise; every rent change is a new ledger row, never an in-place edit.

## 4. Roadmap

Estimates are in **build sessions** (one focused Devin session ≈ 1–2 weeks of a human dev). They assume I do the implementation and you review.

### Phase 0 — Foundations (1 session)
Monorepo, CI (lint/typecheck/test), Postgres schema + migrations, auth (OTP + email, owner/tenant roles, RBAC), S3 uploads, deploy staging.
**Exit**: owner can sign up, create a PG, log in; tenant can sign up.

### Phase 1 — Owner ERP MVP (2–3 sessions) ← the paying product
1. Rooms/beds/occupancy + tenant records + tenancy assignment.
2. Rent cycle: auto invoice generation job, electricity meter readings, extra charges, payment recording (manual + Razorpay link), pending-rent view, PDF invoices.
3. Expenses + dashboard (revenue, collected vs pending, occupancy %, expense breakdown).
4. Complaints inbox with statuses + media.
5. Notifications: WhatsApp/SMS rent reminders + broadcast to all/room/selected.
**Exit**: a real PG can stop using Excel. This is what you demo to sell.

### Phase 2 — Tenant App (1–2 sessions)
Tenant dashboard (rent due, invoices, electricity, agreement PDF, notices), complaint raise with media, push notifications, UPI payment from app, profile + ID proof.
**Exit**: owner onboards tenants; tenant pays and complains in-app. Closes the loop that makes owners stick.

### Phase 3 — Marketplace v1 (1–2 sessions)
PG profile publishing from ERP, explore page with filters + geo search, PG detail page with the transparency breakdown, enquiry/booking-request flow routed to owner inbox, SEO/sitemap for city+locality pages.
**Exit**: inbound tenant demand — your main owner acquisition hook.

### Phase 4 — Employees, multi-property, subscription billing (1 session)
Employee profiles/attendance/salary, multi-PG switcher + consolidated dashboard, your own SaaS subscription with bed-count metering and plan limits, onboarding-fee invoicing.
**Exit**: you can charge, enforce plans, and serve multi-property owners.

### Phase 5 — Reels + AI layer (1–2 sessions)
Short-video upload/transcode + vertical feed with CTAs; promotions/featured listings; AI profit suggestions (pricing, occupancy, expense reduction) grounded in the analytics SQL; AI PG matching + roommate compatibility scoring.
**Exit**: the differentiation in the deck becomes real, on top of data you already have.

### Phase 6 — Hardening & scale (ongoing)
Reconciliation reports, refunds, data export, audit log UI, role permissions polish, load testing, monitoring/alerting, backups + restore drill, Play Store/App Store release.

### Later (post-PMF, from the vision slide)
Digital rental agreements with e-sign, smart access/locks, embedded finance (deposit financing, rent-on-credit), tenant credit profiles.

## 5. Sequencing rationale
The business plan's revenue comes from owners, and owner lock-in comes from rent + complaints + tenant adoption. So ERP → tenant app → marketplace, not marketplace-first: a marketplace with no supply and no operational data has no moat, whereas the ERP gives you the listings, pricing, and occupancy truth that make the marketplace and AI credible.

## 6. Decisions I need from you before Phase 0
1. **Stack**: OK with Next.js + NestJS + Postgres + React Native/Expo, or do you already have a stack/codebase/wireframes to match?
2. **Tenant app on day one**: native (Expo) or mobile PWA to save a session?
3. **Payments**: Razorpay account available? Rent collection through the platform (you hold/route funds — needs escrow/settlement thinking) vs owner's own UPI with recording only?
4. **WhatsApp**: do you have (or want) a WhatsApp Business API provider? It strongly affects reminder adoption.
5. **Pilot scope**: which 1–3 real PGs will pilot Phase 1, and what are their actual rent/electricity rules (per-unit rate, shared meters)? This drives the billing model more than anything else.
6. **Repo**: `kritikagupta863863-netizen/Nexstay` is currently empty — should I scaffold Phase 0 there next?
