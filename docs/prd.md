# PRODUCT REQUIREMENTS: DT Salona

> **This is the file that changes for each product.** Keep the same section headings.

---

## 1. Vision

**DT Salona** is a cloud platform for salons, spas and beauty businesses in Bangladesh. **From appointment to payment, customer to employee, manage your entire salon from one platform.** It gives every business a professional website with online booking, and runs the calendar, billing counter, customer CRM, memberships and loyalty, staff and commission, inventory, accounts and marketing behind it, across one branch or a whole chain.

It answers the questions every salon owner asks and cannot answer today:

- Who is booked today, with whom, and where are the empty slots?
- Which customers have not come back, and how do I win them back?
- How much did each stylist and each branch earn, and what commission is due?
- Which products are running out, and how much are we wasting?
- Is the cash in the drawer what the system says it should be?
- Which branch is making money?

The flow:

```text
Website / Phone / Walk-in -> Appointment -> Service -> Billing -> Payment -> Loyalty / Package
                                                  |                  |
                                           Product usage       Commission, Accounts, Profit
```

Owner: **Dashboard -> Revenue -> Branches -> Staff -> Profit.**
Receptionist: **Calendar -> Check-in -> Billing.**
Customer: **Website -> Online booking -> Reminder -> Visit -> Points / Package.**

**Company:** Datatech BD Ltd, Dhaka.
**Target customers:** beauty salons, hair salons, spas, beauty parlours, makeup studios, men's grooming, multi-branch salons and beauty chains.
**Languages:** English and Bangla (UI, website, documents, messages).
**Mobile app:** not in this project. Stylists and receptionists use the responsive web app; the `/api/v1` is built so a separate mobile app project can use it later.

---

## 2. Business Model & Deployment _(decision)_

**Multi-tenant SaaS.** One application and one shared database serve many businesses (tenants). A tenant has a head office and one or more branches, and a public website on a subdomain or its own domain. Isolation, export and security rules are in `architecture.md` sections 11 and 12.

> A large chain may ask for a dedicated installation. That is a different deployment and pricing model; decide it before you quote one.

### Pricing and plans _(suggested only; no pricing was specified, editable in the platform console)_

| Plan    | Price per month (BDT) | Limits and modules                                                                                                                             |
| ------- | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Starter | about 1,500           | 1 branch, up to 5 staff, 3 users; website, online booking, calendar, billing, CRM, basic reports                                               |
| Growth  | about 4,000           | Up to 3 branches, 25 staff, 10 users; plus memberships and loyalty, packages, gift cards, inventory, staff commission, SMS marketing, accounts |
| Chain   | about 10,000          | Unlimited branches, 100+ staff; plus HR and payroll, WhatsApp, custom domain, advanced reports, API access, priority support                   |

SMS and WhatsApp are sold as prepaid credits. These are business decisions; the code only needs plan limits and module toggles (`PlanGate`) to be configurable per tenant.

### Sales and onboarding flow

1. **Platform console** (Datatech only): create a tenant in minutes, set plan, trial dates and limits, add message credits.
2. **Setup wizard:** salon profile and logo, branches with opening hours, service categories and services (with price, duration and commission), staff and shifts, resources (chairs, rooms), document number formats.
3. **Import wizard:** customers (with balances, packages and points), services, products with opening stock, staff.
4. **Website and booking page:** pick a theme, branding, domain, menus; publish; share the booking link.
5. **Trial:** 30 days with a visible banner; then subscribe. After the trial, 7-day read-only grace, then suspension; data kept for 90 days and exportable.

---

## 3. Roles

| Role                | Scope             | Main needs                                                         |
| ------------------- | ----------------- | ------------------------------------------------------------------ |
| `owner`             | All branches      | Dashboard, revenue, profit, approvals, settings                    |
| `branch-manager`    | Own branch        | Calendar, staff, stock, daily cash, approvals, reports             |
| `receptionist`      | Own branch        | Calendar, check-in, billing, customer service                      |
| `stylist`           | Own appointments  | Today's schedule, notes, complete service, own performance         |
| `cashier`           | Own branch        | Billing counter, collections, daily cash close                     |
| `inventory-manager` | Assigned branches | Purchases, stock, usage, transfers, counts                         |
| `accountant`        | All branches      | Vouchers, bank, supplier and staff payments, reports, period close |
| `hr-manager`        | All branches      | Employees, attendance, leave, shifts, payroll                      |
| `marketing-manager` | All branches      | Segments, campaigns, coupons, offers, website content              |
| `web-editor`        | Website           | Pages, services content, gallery, blog, careers, SEO               |

Platform role: `super-admin` (Datatech staff), not tied to a tenant.
**Customers** can log in with mobile OTP to see their own appointments, packages, points and invoices.
Prices, costs, commissions, payroll, profit and sensitive customer notes are visible only with their permissions (`services.view-cost`, `commission.view`, `payroll.view`, `reports.profit`, `customers.view-sensitive`).

---

## 4. Modules

### A. Tenants, Plans & Platform Admin (Datatech)

Create and suspend tenants, plans, module toggles, limits, trial dates, message credits, usage, impersonation with audit log, import monitor, and a trials-ending list.

### B. Public Website & Online Booking

- **Pages:** home, about, services (by category), pricing, packages, team, gallery (before/after with consent), offers, membership plans, branches with map and hours, appointment booking, contact, blog, careers with applications, privacy and terms, custom pages.
- **Builder:** theme and branding, menus, sections with ordering, banners, bilingual content, per-page SEO fields, redirects, sitemap, custom domain with TLS.
- **Online booking:** branch, services or package, preferred staff or "any", slot from the slot engine, details and OTP, confirmation. **A booking automatically creates or updates the customer in the CRM and appears on the calendar.** Optional deposit, waitlist, reschedule and cancel by the customer within the allowed window.
- **Lead capture:** contact, membership and careers forms with anti-spam.
- **Analytics hooks:** UTM capture on bookings, optional tag manager and pixel IDs.

### C. Customer CRM

- **Profiles:** name, mobile, email, gender, birthday, anniversary, address (optional), source, preferred branch and staff, allergies and sensitive notes (permission), general notes, tags, photo and before/after photos (consent), consultation forms.
- **History:** visits, services with the staff and products used, invoices, packages, wallet, points, dues, messages, no-shows.
- **Preferences:** favourite services, products, staff, and communication channel.
- **Segmentation:** inactive (default 60 days), VIP, new, birthday and anniversary this week, membership or package expiring, and custom filters; saved segments with live counts.
- **Duplicate handling:** merge by mobile; a repeat online booking updates the existing customer.
- **Import and export** with permissions and access logs.

### D. Appointment Management

- **Calendar:** day, week, by staff, by resource, by branch; drag and drop; colored status blocks with labels.
- **Booking sources:** online, phone, walk-in (with a queue), and staff-created; multi-service appointments with different staff per service.
- **Slot engine:** opening hours, holidays, staff shifts and leave, skills, service duration plus buffer, resources.
- **Reschedule, cancellation and no-show tracking** with configurable rules and optional deposits.
- **Reminders:** confirmation, 24-hour and 2-hour reminders, thank-you and review request, by SMS, WhatsApp or email.
- **Waitlist** with notification when a slot opens.
- **Check-in** and in-service timers; the completed appointment becomes a draft invoice.

### E. Service Management

- **Categories:** hair, facial, makeup, spa, skin, nails, grooming, bridal and others (configurable).
- **Each service:** name (two languages), category, description, price (default and per branch), duration, buffer, gender (optional), required skill, required resource, **staff commission rule**, **required products** (recipe), branch availability, online-bookable flag, image.
- **Service packages:** a bundle of services with a package price, sessions and validity.
- **Price lists** with effective dates; seasonal and happy-hour pricing rules (simple).

### F. Membership & Loyalty

- **Membership plans:** fee, duration, benefits (discount percent, free services, extra points, priority booking), auto-renewal reminders.
- **Packages (prepaid sessions)** sold from the counter or online (later), with remaining sessions, expiry and eligible services.
- **Loyalty points:** earning and redemption rules, expiry, tiers (for example Silver, Gold, VIP).
- **Discounts and coupons:** by percent or amount, validity, usage limits, branch and service scope.
- **Gift cards** (physical or digital) with code, value, expiry and balance.
- **Wallet:** prepaid customer balance with top-up and use.
- **Referral rewards** for referrer and new customer.

### G. Billing & Payments

- Billing counter built from completed appointments or walk-in sales; services and retail products; discounts, coupons, tips, split payments (cash, card or POS, mobile banking with TrxID, bank transfer, wallet, gift card, package, points).
- Customer dues with later collection receipts.
- Invoice print and digital copy; void and credit note with approval; refunds.
- Daily cash close per branch.

### H. Staff / HR

- **Employees:** profile, documents, role, branch, skills, services they perform, shift pattern.
- **Attendance:** web check-in, manual entry, device file import; shifts and rosters; holidays.
- **Leave:** types, balances, requests, approval.
- **Salary:** structures, allowances, deductions, advances, monthly payroll with approval, lock, payslips, journal posting.
- **Commission and incentives:** per service and per product rules, targets and slabs, monthly statements with approval and lock.
- **Performance:** services done, revenue, rebooking rate, rating, utilization, upsell; leaderboard.
- **Recruitment:** job posts from the website careers page and applicants.

### I. Inventory

- **Products:** retail products and consumables, units, barcodes, suppliers, reorder levels, optional batch and expiry.
- **Purchases:** suppliers, purchase orders, goods receipt, returns, supplier bills.
- **Stock:** per branch, movements ledger, transfers between branches, stock counts with approval.
- **Usage:** service recipes deduct stock on completion; actual usage can be recorded.
- **Wastage:** recorded with reason and approval above a limit.
- **Alerts:** low stock and near expiry.
- Retail sales through the billing counter.

### J. Accounts

- Chart of accounts template, fiscal years, periods with locking.
- **Automatic postings** from invoices, payments, dues, wallet, gift cards, packages, purchases, expenses, payroll and commissions.
- Vouchers (receipt, payment, journal, contra), bank and cash books, manual bank reconciliation.
- **Expenses**, supplier payments, employee payments, with approval above a limit.
- **Customer dues** and supplier payables with aging.
- **Daily cash** per branch.
- **Reports:** sales, payments by method, expenses, profit and loss, branch profitability, cash flow summary, receivables, payables, trial balance.
- VAT is configurable and off by default; confirm applicable rules with the client's accountant.

### K. Marketing

- **Segments** (inactive 60 days, VIP, birthdays, expiring packages and memberships, custom).
- **Campaigns** by SMS, WhatsApp and email with templates, optional coupon, schedule, cost estimate first, and result tracking (sent, delivered, bookings, revenue).
- **Automations:** birthday and anniversary wishes, "We miss you" at the inactive threshold (for example "We miss you! Get 15% off your next visit."), post-visit thank-you and review request, membership and package renewal reminders.
- Coupons and offers shown on the website.
- Review and feedback collection.

### L. Multi-Branch

Head office with branches (Dhaka, Branch A, Branch B, Branch C and more). Management sees all branches: consolidated dashboard, branch comparison, per-branch settings (hours, prices, availability, staff, stock). Customers, loyalty, packages and gift cards work across branches by tenant setting. Branch managers see only their branch.

### M. Tasks, Approvals & Notifications

Generic tasks, one approval engine (discounts, voids, refunds, price changes, expenses, leave, payroll, commission payout, stock adjustment, day unlock), in-app bell, SMS and email notifications.

### N. Dashboard & Reports

Dashboard KPIs per section 5 with drill-down, today's appointments and utilization, branch comparison, top services and staff, inactive customers, low stock, cash position, and alerts. Reports with PDF and Excel export follow `design.md` section 17.

### O. Data Import, Onboarding & API

Import wizards (dry run, row errors, safe re-run), onboarding checklist and demo data, and a documented `/api/v1` ready for a future mobile app.

---

## 5. Business Rules & KPI Definitions

### KPI definitions _(fixed; every screen and report must use these)_

| Metric                      | Definition                                                                                                                                                                                                                                                                                                |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Revenue**                 | Sum of finalized invoice totals in the period, excluding tax and tips, net of discounts and credit notes. Prepaid sales (packages, gift cards, wallet top-ups, memberships) are reported separately as **prepaid sales** and recognized as revenue when used (package sessions at their allocated value). |
| **Collections**             | Payments received in the period by method (cash, card, mobile banking, bank, wallet and gift card shown as redemptions).                                                                                                                                                                                  |
| **Appointments**            | Count of appointments with a start time in the period, by status; **completed** appointments are the basis for service KPIs.                                                                                                                                                                              |
| **Utilization**             | Booked (completed plus in-service) minutes divided by available staff minutes (shift time minus leave and breaks) in the period.                                                                                                                                                                          |
| **No-show rate**            | No-show appointments divided by appointments that were due (completed plus no-show).                                                                                                                                                                                                                      |
| **New customers**           | Customers whose first completed visit falls in the period.                                                                                                                                                                                                                                                |
| **Retention (repeat rate)** | Customers with a completed visit in the period who also had a completed visit in the previous 90 days, divided by customers with a completed visit in the previous 90 days.                                                                                                                               |
| **Average ticket**          | Revenue divided by number of finalized invoices with at least one service or product line.                                                                                                                                                                                                                |
| **Expenses**                | Posted expense and payment vouchers in the period (operating costs, supplier purchases expensed, payroll), excluding transfers and refunds.                                                                                                                                                               |
| **Profit**                  | Income statement result from posted journals: revenue plus recognized prepaid usage minus cost of products used, commissions, payroll and expenses. Branch profit uses the same method per branch.                                                                                                        |
| **Inactive customer**       | A customer with no completed visit in the last N days (default 60, configurable), who has had at least one visit.                                                                                                                                                                                         |

Dashboard totals must equal report totals and ledger totals exactly.

### Rules

| Topic                                    | Rule                                                                                                                                                                           |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Double booking**                       | A staff member or resource cannot have overlapping appointment items. Enforced by locking and re-checking the slot in a transaction. Concurrency is tested.                    |
| **Appointment status**                   | Changes only through `ChangeAppointmentStatusAction` with history. No-show and late cancellation rules are configurable per tenant.                                            |
| **Online booking**                       | Needs OTP verification, creates or merges the customer by mobile, and respects slots, hours and leave. A customer over the no-show threshold may be required to pay a deposit. |
| **Pricing**                              | A service price is taken from the branch price list at booking and stored on the appointment item and the invoice line. Price changes never alter finalized invoices.          |
| **Invoices**                             | Finalized invoices are immutable. Corrections use credit notes or voids with a reason and approval. Numbers are sequential per branch series without gaps.                     |
| **Payments**                             | Split payments must sum to the total. Partial payment creates an explicit due. Tips are recorded separately from revenue.                                                      |
| **Wallet, packages, points, gift cards** | Each is an append-only ledger; balances are the sum of entries and never typed. Voiding an invoice reverses the entries it created. Expiry is handled by an idempotent job.    |
| **Coupons**                              | Valid within dates, usage limits and scope; single-use codes are redeemed once under a lock.                                                                                   |
| **Commission**                           | Calculated per invoice item at finalization from the service or product rule; reversed on void or credit; statements are approved and locked before payroll.                   |
| **Stock**                                | Changes only through immutable movements; service usage is posted when the service completes or the invoice is finalized; negative stock is blocked unless allowed.            |
| **Daily cash**                           | A branch closes each day by counting cash; differences need approval; a closed day cannot be edited without an approved unlock.                                                |
| **Accounting**                           | All postings go through `PostJournalEntryAction`; journals balance; posted entries are immutable; locked periods reject postings.                                              |
| **Customers across branches**            | Customers are tenant-wide; each visit records its branch.                                                                                                                      |
| **Marketing consent**                    | Marketing messages respect opt-out; transactional messages are always allowed; bulk sends show segments and cost first and consume credits atomically with logs.               |
| **Photos and notes**                     | Sensitive notes and photos need permission; photos on the website need recorded consent.                                                                                       |
| **Website data**                         | Only publishable data appears; changes purge caches within a minute.                                                                                                           |
| **Records**                              | No hard deletes of customers with history, invoices, payments, ledgers, journals or employees with payroll.                                                                    |
| **Tenant isolation**                     | No query may return another tenant's data. Tested for every model and endpoint.                                                                                                |
| **Plan limits**                          | Branches, staff, appointments per month, users, storage and modules are enforced centrally with a clear message and an upgrade path.                                           |
| **Regulatory**                           | VAT, gift card and membership terms, refunds, and data protection must be confirmed with a lawyer and accountant. Templates are editable and carry no legal guarantee.         |

---

## 6. Hardware

- Desktop, laptop or tablet at reception; phones for stylists and managers.
- Optional: 80mm thermal printer for receipts, barcode scanner for retail and stock, card or POS terminal (payments are recorded manually in v1), tablet for customer check-in or consultation form signing.

---

## 7. Non-Functional Requirements

- **Localization:** UI, website, documents and messages in `en` and `bn`.
- **Performance:** calendar day view loads in under 1 second for 30 staff; slot search under 500 ms; invoice finalize under 1 second; dashboard under 3 seconds for a tenant with 100,000 customers, 20 branches and 3 years of data; public pages LCP under 2.5 seconds on mid-range phones.
- **Scale targets:** a chain with 20 branches, 300 staff, 200,000 customers and 2 million invoice lines remains responsive using indexes and aggregated tables.
- **Security:** HTTPS only, encrypted sensitive fields, private file storage with signed URLs, access logs for sensitive views and exports, rate limits, idle timeout, encrypted backups, tenant isolation.
- **Audit:** activity log for price and discount changes, voids, refunds, wallet, points and gift card adjustments, stock adjustments, day unlocks, role changes and exports.
- **SEO and accessibility:** Lighthouse SEO and accessibility scores of 95 and 90 or more on public pages.
- **Availability:** target 99.5% monthly; planned maintenance outside working hours.
- **Browsers:** latest Chrome, Edge, Safari and Firefox for the public site; latest Chrome and Edge (tablet browsers included) for the back office.

---

## 8. Screen List (per role)

| Role                    | Screens                                                                                              |
| ----------------------- | ---------------------------------------------------------------------------------------------------- |
| owner                   | Dashboard, branch comparison, revenue and profit, staff performance, approvals, settings, access log |
| branch-manager          | Calendar, staff roster, stock, daily cash, approvals, branch reports                                 |
| receptionist            | Calendar, walk-in queue, check-in, customer search and profile, billing counter, messages            |
| stylist (mobile)        | Today's timeline, appointment card, complete service, my performance, attendance and leave           |
| cashier                 | Billing counter, collections of dues, daily cash close                                               |
| inventory-manager       | Products, purchases, stock, transfers, usage and wastage, counts, low-stock alerts                   |
| accountant              | Vouchers, bank and cash books, supplier and staff payments, reports, period close                    |
| hr-manager              | Employees, attendance, leave, shifts, payroll, commission statements, recruitment                    |
| marketing-manager       | Segments, campaigns, automations, coupons, offers, reviews                                           |
| web-editor              | Pages, services content, gallery, blog, careers, SEO, redirects                                      |
| customer (account area) | My appointments, packages, points and gift cards, invoices, referral                                 |
| platform (Datatech)     | Tenants, plans and trials, usage, credits, impersonation, import monitor                             |

---

## 9. Out of Scope

- Native mobile apps (separate project; this project provides the API).
- Online gateway payments (bKash, Nagad, cards) at online booking; deposits are recorded in person or by mobile banking with verification. Gateway integration is a later module.
- Integrated card or POS terminals (payments are entered manually).
- Electronic signatures (consultation forms are signed on screen as an image or uploaded as scans).
- Biometric attendance device drivers (device file import is included).
- Full ERP procurement and manufacturing.
- Statutory tax filing and deep payroll tax compliance.
- Marketplace or aggregator listings of many salons.
- Video consultations and AI skin analysis.
- Dedicated per-client installations unless agreed separately.

---

## 10. Key User Stories & Acceptance Criteria

1. **Online booking.** _As a customer, I book a service on the website._
    - Done when: slots shown are real, an OTP confirms my mobile, a confirmation is sent, the appointment appears on the calendar, and I exist in the CRM (merged if I already did).
2. **Reception booking.** _As a receptionist, I book a customer by phone._
    - Done when: it takes under 30 seconds, only valid slots are offered, multi-service appointments with different staff work, and two simultaneous bookings of the same slot result in one appointment (tested).
3. **Walk-in.** _As a receptionist, I register a walk-in._
    - Done when: the customer is placed in the queue or the next free slot and the stylist sees them.
4. **Reschedule and cancel.** _As a customer or staff, I change an appointment._
    - Done when: history keeps the old time, reminders update, rules and fees apply, and the waitlist is offered the freed slot.
5. **No-show.** _As a manager, I control no-shows._
    - Done when: no-shows are marked, counted per customer, and a threshold can require a deposit.
6. **Reminders.** _As an owner, fewer customers forget._
    - Done when: confirmation and reminders go on schedule, cancelled appointments send none, and logs show cost and delivery.
7. **Serve and bill.** _As a stylist and cashier, we finish a service._
    - Done when: the stylist completes the service and confirms products used, the invoice opens prefilled, package and membership benefits apply, split payments sum to the total, and the invoice prints.
8. **Membership and package.** _As a cashier, I sell and redeem._
    - Done when: a package shows sessions left, each use writes a ledger entry with the staff member, expiry is enforced, and a refund follows the approved rule.
9. **Loyalty and gift cards.** _As a customer, I earn and spend._
    - Done when: points earn and expire by rule, a void reverses points, a gift card cannot be guessed or overspent, and balances equal the ledger sum (tested).
10. **Commission.** _As an owner, I pay staff fairly._
    - Done when: commission per item matches hand-calculated examples for every rule type, a void reverses it, and an approved statement is locked.
11. **Inventory.** _As an inventory manager, I avoid stockouts._
    - Done when: purchases update stock and cost, service usage deducts by recipe, wastage needs a reason, transfers between branches record differences, and low-stock alerts fire once per day.
12. **Daily cash.** _As a branch manager, I close the day._
    - Done when: expected versus counted cash is shown, a difference needs approval, and the closed day cannot be edited without an approved unlock.
13. **Accounts.** _As an accountant, my books are right._
    - Done when: all postings balance, wallet, gift cards and packages appear as liabilities until used, periods lock, and the trial balance equals the ledgers.
14. **Win-back campaign.** _As a marketing manager, I recover lost customers._
    - Done when: the segment of customers inactive 60 days shows an exact count, the cost is shown before sending, opted-out customers are excluded, a coupon is attached, and bookings and revenue are tracked.
15. **HR and payroll.** _As an HR manager, I run payroll._
    - Done when: attendance, leave and approved commission feed the salary sheet, approval locks the month, payslips print, and the journal posts.
16. **Multi-branch.** _As an owner, I see my whole chain._
    - Done when: the dashboard consolidates branches, a branch manager sees only their branch (tested), customers are recognized across branches, and per-branch prices apply.
17. **Website editing.** _As a web editor, I publish services and offers._
    - Done when: bilingual content, gallery with consent, SEO fields and prices are saved, and the page is live within a minute.
18. **Dashboard.** _As an owner, my numbers agree._
    - Done when: KPIs equal reports and ledgers (tested) and nightly reconciliation shows no differences.
19. **Import.** _As Datatech support, I move a salon in a day._
    - Done when: a dry run reports row errors, the real import is repeatable without duplicates, and balances, packages and points appear in their ledgers.
20. **Tenant isolation.** _As a business, my data is never visible to another._
    - Done when: automated tests prove every model and endpoint returns 404 across tenants, including the website and booking API.
21. **Subscription and trial.** _As Datatech, I run a trial and bill the tenant._
    - Done when: a banner shows days left, limits are enforced, expiry goes to read-only for 7 days and then suspension, and data can be exported.

---

## 11. Phasing

See `roadmap.md`. Build phase by phase; do not implement later phases early. The first sellable release is Phases 0 to 7 (setup, services, CRM, calendar, billing, website with online booking, reminders and basic dashboard); the rest are upgrades by plan.
