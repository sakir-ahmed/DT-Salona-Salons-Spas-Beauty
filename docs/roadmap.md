# ROADMAP: DT Salona

> The AI builds **one phase at a time**, runs tests, and stops. Each phase must satisfy the Definition of Done in `architecture.md`.
> **First sellable release = Phases 0 to 7** (setup, services and staff, customer CRM, calendar, billing counter, website with online booking, reminders and the owner dashboard). Phases 8 to 13 are upgrades for Growth and Chain plans.

Mark finished phases with `[x]`.

---

## Phase 0: Foundation, Tenancy & UI Kit

- [ ] Docker Compose (nginx, php-fpm with Chromium, mysql, redis, queue, scheduler), `.env.example`
- [ ] Multi-tenancy: `Tenant`, `ResolveTenant`, `ResolveWebsite`, `BelongsToTenant`, `BelongsToBranch`, tenant-aware jobs, prefixed cache and storage
- [ ] Architecture test that fails when a tenant-owned model lacks the trait
- [ ] Inertia + Vue 3 + Tailwind v4 tokens (`design.md`), Sonner, dark mode, SSR on public pages only
- [ ] Localization middleware, `en` / `bn` files, language switcher
- [ ] Staff auth (username / email / mobile), OTP service, rate limits, idle timeout
- [ ] Spatie Permission with teams, per-tenant role seeding, platform `super-admin`
- [ ] Activity log, access log, encrypted backups, tenant export command
- [ ] Minimal `/platform` console: create tenant in under 5 minutes, plan, limits, trial dates, message credits
- [ ] `PlanGate` skeleton (limits, modules, trial and suspension rules)
- [ ] **Shared UI kit** from `design.md` section 22 with `AppIcon` and `NoEmojiTest` in CI
- [ ] OpenAPI docs generator at `/api-docs`
- [ ] Hidden `/admin/ui-kit` showcase page that renders every component in light, dark, `en` and `bn`

**Acceptance:** `migrate:fresh --seed` works; two seeded tenants cannot see each other's data (tested); each role sees only its menu; the UI-kit page renders.

## Phase 1: Setup, Branches, Services, Staff & Resources

- [ ] Setup wizard, salon profile, branches with opening hours and holidays, document number series
- [ ] Service categories and services (price, duration, buffer, skill, resource, online-bookable), per-branch price and availability, price lists with effective dates
- [ ] Staff profiles, skills and the services they perform, shifts and rosters, leave blocks
- [ ] Resources (chairs, beds, rooms) per branch
- [ ] Service import (dry run, row errors, safe re-run)

**Acceptance:** per-branch price and availability resolve correctly (tested); a service can only be assigned to staff with the skill; branch-scoped users never see other branches (tested); importing the same file twice creates no duplicates.

## Phase 2: Customer CRM

- [ ] Customers (profile, birthday, anniversary, preferences, notes, tags), duplicate detection and merge by mobile
- [ ] Sensitive notes and photos with permission, consent records, access log
- [ ] Customer 360 page, visit and service history placeholders fed later by appointments and invoices
- [ ] Segments: new, VIP rules, inactive N days, birthday and anniversary this week (idempotent refresh job), saved segments with live counts
- [ ] Customer import with balances, packages and points opening entries (ledger rows, not typed balances)

**Acceptance:** repeat mobile never creates a duplicate; a user without `customers.view-sensitive` never receives sensitive data from any route or API (tested); segment counts equal the matching customers exactly.

## Phase 3: Appointments & Calendar

- [ ] `AvailabilityService` slot engine (hours, holidays, shifts, leave, skills, duration, buffer, resources)
- [ ] `BookAppointmentAction`, appointment items with their own staff and times, `ChangeAppointmentStatusAction` with history
- [ ] Calendar day, week, by staff, by resource and by branch; drag and drop; walk-in queue; check-in
- [ ] Reschedule, cancellation and no-show rules, per-customer no-show count
- [ ] Waitlist

**Acceptance:** two simultaneous bookings of the same staff slot produce exactly one appointment (concurrency test); no slot is offered during leave, breaks, closed hours or resource conflicts (tested); illegal status transitions are rejected; a reschedule keeps history.

## Phase 4: Billing Counter, Payments & Daily Cash

- [ ] Draft invoice from a completed appointment or walk-in sale; retail product lines (stock hooks arrive in Phase 9)
- [ ] Discounts with reasons and limits, tips, split payments, partial payment as customer due, later due collection with receipt
- [ ] Immutable finalized invoices, void and credit note with approval, gapless numbering per branch
- [ ] Approval engine (inbox, reminders, escalation) first used for discounts and voids
- [ ] Invoice PDF and 80mm print, QR copy
- [ ] Daily cash close with difference approval and day lock

**Acceptance:** payments always sum to the invoice total (tested); invoice numbers are gapless under concurrency; a finalized invoice cannot be edited; a closed day rejects edits until an approved unlock; cash expected equals the sum of cash payments and cash expenses.

## Phase 5: Public Website & Online Booking

- [ ] `ResolveWebsite` host mapping, subdomain and custom domain with verification and TLS
- [ ] Pages, menus, sections, banners, blog, careers, team, gallery with consent, offers, branches with map, custom pages (bilingual)
- [ ] Services, pricing and packages pages fed from the service catalog
- [ ] Online booking flow using `BookAppointmentAction`: OTP, Turnstile, idempotent submission, customer create or merge, confirmation
- [ ] Media pipeline (WebP, alt text), SEO (meta, sitemap, `robots.txt`, redirects, JSON-LD, `hreflang`), per-tenant cache with purge
- [ ] Branding with accent contrast check; privacy and terms pages

**Acceptance:** an unknown host returns 404; one tenant's site never shows another's content (tested); every online booking appears on the calendar and in the CRM (tested); the same submission sent twice creates one appointment; Lighthouse SEO 95+ and accessibility 90+ on the seeded demo site; LCP under 2.5 seconds on the mobile profile.

## Phase 6: Messaging, Reminders & Tasks

- [ ] `MessageChannel` with SMS, email and WhatsApp, templates, credits wallet, logs, opt-out
- [ ] Appointment confirmation, reminders, thank-you and review request jobs (idempotent)
- [ ] Generic tasks and "my tasks"
- [ ] In-app notification bell and daily owner summary

**Acceptance:** reminders run twice and send once; a cancelled appointment sends none; bulk sends show segments and credits first and fail safely at zero credits; credits and logs never diverge (tested).

## Phase 7: Dashboard & Reports

- [ ] Owner and branch dashboards with the KPIs from `prd.md` section 5 and drill-down
- [ ] Reports with PDF and Excel export following `design.md` section 17: revenue, payments by method, appointments, no-shows, utilization, customers, retention, top services
- [ ] Aggregated tables and queued long reports
- [ ] Onboarding checklist and demo data

**Acceptance:** dashboard totals equal report totals and ledger totals exactly (tested); the dashboard loads in under 3 seconds on a seeded tenant with 100,000 customers and 3 years of data. **Profit shows revenue and recorded expenses only until Phase 10. This completes the first sellable release.**

## Phase 8: Membership, Packages, Loyalty, Gift Cards & Coupons

- [ ] Wallet, package, points and gift card **append-only ledgers**
- [ ] Membership plans and benefits, renewal reminders, expiry job
- [ ] Packages (prepaid sessions) with redemption linked to staff, refunds by rule with approval
- [ ] Loyalty rules, tiers, expiry, reversal on void
- [ ] Coupons and offers with limits and scope; gift cards with random codes
- [ ] Referral rewards
- [ ] Balances shown on the customer banner and billing counter

**Acceptance:** every balance equals the sum of its ledger (tested); a void reverses what the invoice created; a single-use coupon redeems once under concurrency; gift cards cannot be overspent; expiry jobs are idempotent.

## Phase 9: Inventory

- [ ] Products (retail and consumable), suppliers, purchase orders, goods receipt, returns
- [ ] `PostStockMovementAction`, per-branch balances under lock, weighted average cost
- [ ] Service recipes with automatic usage posting; actual usage and wastage with approval
- [ ] Transfers between branches, stock counts with variance review, low-stock and expiry alerts
- [ ] Retail sales through the billing counter

**Acceptance:** for every product and branch, balance equals the sum of movements (reconciliation test, including concurrency); service completion deducts the recipe; negative stock is blocked unless allowed; weighted cost matches hand-calculated examples.

## Phase 10: Accounts

- [ ] Chart of accounts, fiscal years, periods with locking
- [ ] `PostJournalEntryAction` and automatic postings from invoices, payments, dues, wallet, gift cards, packages, purchases and expenses
- [ ] Vouchers, cash and bank books, manual bank reconciliation, expenses, supplier and staff payments with approval
- [ ] Reports: profit and loss, branch profitability, cash flow summary, receivables, payables, trial balance

**Acceptance:** every journal balances (tested); posted entries cannot be edited; a locked period rejects postings; wallet, gift card and package liabilities equal their ledgers; branch profit equals journal results.

## Phase 11: Staff HR, Commission & Payroll

- [ ] Employee records and documents, attendance (web check-in, manual, import), shifts and holidays, leave and balances
- [ ] Commission and incentive rules (per service, per product, slabs, targets), per-item calculation at finalization, reversal on void
- [ ] Monthly commission statements with approval and lock; performance dashboards and leaderboard
- [ ] Payroll: structures, advances, monthly run, approval, lock, payslips, journal posting
- [ ] Recruitment from website careers

**Acceptance:** commission matches hand-calculated examples for every rule type; a void reverses commission; an approved statement and payroll month cannot be edited; salary journal balances; payroll data is visible only to permitted roles.

## Phase 12: Marketing Campaigns & Automations

- [ ] Campaign builder on segments with channel, template, coupon, schedule, cost estimate first
- [ ] Automations: birthday, anniversary, "We miss you" at the inactive threshold, renewal reminders
- [ ] Attribution of bookings and revenue to campaigns, reviews and feedback collection

**Acceptance:** opted-out customers are never messaged; a send is idempotent per campaign and customer; the cost shown equals the credits used; attribution counts only bookings inside the window.

## Phase 13: Subscription, Billing, Polish & Hardening

- [ ] Platform console: plans, module toggles, limits, suspend or resume, usage, impersonation with audit
- [ ] Trial banner, 7-day read-only grace, suspension, scheduled retention, subscription records, message credit top-ups
- [ ] API review for the future mobile app, OpenAPI examples, contract tests that keep `v1` stable
- [ ] Security review (access logs, encryption, backup restore drill), accessibility pass, performance check with 100,000 customers and 20 branches
- [ ] User guide (English and Bangla), onboarding guide for Datatech staff, deployment guide (including custom domain TLS)

**Acceptance:** full test suite green, Pint and Larastan clean, a restore drill passes, nightly reconciliation shows zero differences on the performance dataset, limits for branches, staff and users are enforced everywhere (tested), and a new salon can be set up with its website live and first online booking received within one day.

---

## Later Modules (not in this project scope)

Mobile apps for stylists and customers, online payment at booking (bKash, Nagad, cards), integrated card terminals, biometric attendance drivers, marketplace listings, video consultation, statutory tax filing.
