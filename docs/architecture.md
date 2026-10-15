# ARCHITECTURE & CODING RULES: DT Salona

> Shared engineering rules for Datatech products, adapted for DT Salona (multi-tenant SaaS for salons, spas and beauty chains: public website with online booking, customer CRM, appointments, services, memberships and loyalty, staff and HR, inventory, accounts, marketing, multi-branch). Sections 11 to 18 are specific to this product.
> If this file conflicts with the product's `prd.md`, **the PRD wins** for business rules and roles. This file wins for code structure.

---

## 1. Tech Stack

| Area                    | Choice                                                                                                                                                                        |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Framework               | Laravel 13.x (latest stable), PHP 8.3+. Check with `composer show laravel/framework`.                                                                                         |
| Database                | MySQL 8                                                                                                                                                                       |
| Cache / Session / Queue | Redis (`CACHE_STORE`, `SESSION_DRIVER`, `QUEUE_CONNECTION` = `redis`)                                                                                                         |
| Frontend                | Inertia.js v2, Vue 3 (`<script setup>`), Tailwind CSS v4, shadcn-vue (Radix Vue)                                                                                              |
| Toasts                  | `vue-sonner`                                                                                                                                                                  |
| Auth                    | Staff: username / email / mobile login. Customers: mobile OTP (separate `customer` guard, booking and account pages only). Spatie Permission with **teams** (tenant as team). |
| Real-time               | Not required in v1. Use page refresh and light polling; notifications go through the in-app bell, SMS and email.                                                              |
| Audit                   | `spatie/laravel-activitylog`                                                                                                                                                  |
| Backup                  | `spatie/laravel-backup` (daily, via scheduler)                                                                                                                                |
| Localization            | `en` and `bn` (`lang/en`, `lang/bn`)                                                                                                                                          |
| SMS                     | BdBulkSMS through `SmsChannel` (sections 5 and 16)                                                                                                                            |
| Testing                 | Pest                                                                                                                                                                          |
| Code quality            | Laravel Pint, Larastan (level 5 minimum)                                                                                                                                      |
| Containers              | Docker Compose: nginx, php-fpm (with Chromium), mysql, redis, queue worker, scheduler                                                                                         |
| Icons                   | Font Awesome Free via `@fortawesome/vue-fontawesome` (tree-shaken SVG), wrapped in `AppIcon`                                                                                  |
| Charts                  | `chart.js` + `vue-chartjs` (one library only)                                                                                                                                 |
| Multi-tenancy           | One app, shared MySQL database, `tenant_id` on every tenant-owned table (section 11)                                                                                          |
| PDF                     | `spatie/laravel-pdf` with the Browsershot (headless Chromium) driver                                                                                                          |
| QR                      | `endroid/qr-code` (receipt and document verification codes)                                                                                                                   |
| Translatable content    | `spatie/laravel-translatable` (English and Bangla content columns for services, packages, pages, posts)                                                                       |
| Images                  | `intervention/image` (WebP derivatives, responsive sizes, watermark option)                                                                                                   |
| Sitemap / SEO           | `spatie/laravel-sitemap`, per-tenant `robots.txt`, JSON-LD helpers                                                                                                            |
| Maps                    | Leaflet with OpenStreetMap-compatible tiles (branch locator; provider in `.env`)                                                                                              |
| Email                   | Laravel Mail with per-tenant SMTP or a shared provider (SES, Mailgun)                                                                                                         |
| WhatsApp                | WhatsApp Business Cloud API (Meta) through `MessageChannel` (section 16)                                                                                                      |
| Excel import / export   | `maatwebsite/excel`                                                                                                                                                           |
| API docs                | `dedoc/scramble` (OpenAPI at `/api-docs`)                                                                                                                                     |
| Bot protection          | Cloudflare Turnstile on public inquiry forms, careers and OTP forms                                                                                                           |

**Rendering**

- Public pages (`Web/` controllers): each tenant's corporate website (home, services, pricing, packages, team, gallery, offers, membership, online booking, blog, careers, contact, and more). Inertia SSR with dynamic meta tags, per-tenant domain and cached output (section 13).
- The customer account area after login (my appointments, packages, points, gift cards) is client-side only.
- Admin panel (`/admin`): client-side SPA, **SSR disabled**. No Filament.

---

## 2. Folder Structure

```text
app/
├── Actions/            # One class = one business operation (CreateOrderAction)
├── Services/           # External integrations + orchestration (SmsService, BkashService)
├── DataTransferObjects/# Only when an Action has 4+ inputs
├── Enums/              # OrderStatus, PaymentMethod ...
├── Http/
│   ├── Controllers/
│   │   ├── Admin/      # Inertia controllers for the SPA (booking and collection desks included)
│   │   ├── Api/V1/     # JSON API (see api.md)
│   │   └── Web/        # Public SSR pages
│   ├── Requests/       # Form Requests (all validation lives here)
│   └── Resources/      # API Resources
└── Models/             # Relationships, scopes, casts, LogsActivity. No business logic.
```

**No Repository layer.** Eloquent is the data layer. Add an interface only when there are two or more real implementations. In this product that applies to one place: `MessageChannel` (SMS, email, WhatsApp) in `app/Services/Messaging/`. There are no payment gateways in v1 (collections are cash, cheque, bank and mobile money entered by staff, or submitted by customers as payment proof). Approval workflows use one engine, not one class per request type.

---

## 3. Core Rules

1. **Thin controllers.** Validate (Form Request) → call an Action → return a response.
2. **Admin controllers and API controllers call the same Actions.** Never duplicate business logic.
3. **Strict types.** `declare(strict_types=1);`, typed parameters, return types and properties everywhere.
4. **No over-engineering.** No factories, abstractions or patterns without a concrete need today.
5. **Money:** store as `decimal(12,2)` in BDT. Never use floats. API returns money as strings (`"1250.00"`).
6. **Appointments, invoices, payments, stock movements, wallet, package and points ledgers, journal postings and payroll run inside `DB::transaction()`.** Use `lockForUpdate()` on the staff and resource calendar rows (to stop double booking), on document number counters and on balances.
7. **Tenant and branch scoping:** every tenant-owned model uses `BelongsToTenant`, and branch-owned models also use `BelongsToBranch` (section 11). A user must never read another tenant's data, and non-owner users must never read another branch's data. Test both.
8. **All user-facing text uses translation keys** (`__('orders.created')`), never hard-coded strings.
9. **Secrets only in `.env`**, accessed via `config()`. Never call `env()` outside config files.
10. **No emojis anywhere** (UI, translations, SMS, receipts, seeders, comments, commits). Icons come from Font Awesome through `AppIcon`; see `design.md` section 23.
11. **Eager load relations** to avoid N+1. Add DB indexes on foreign keys and common filters (branch, staff and start time, customer mobile, appointment status, invoice number, product and branch, account and period).
12. **Customer data rules apply everywhere** (section 12): no customer personal data in logs, URLs or error trackers; skin, hair and health notes and photos are sensitive and permission-protected.
13. **Money and time:** amounts use `decimal(12,2)` BDT; service durations are whole minutes; all appointment times are stored in UTC with the branch time zone (`Asia/Dhaka` by default) and shown in branch time.

---

## 4. Roles & Permissions

Spatie Permission with **teams enabled** (`teams => true`, team key `tenant_id`), so every role and assignment belongs to one tenant. Roles are defined in `prd.md` and seeded per tenant when the tenant is created. `super-admin` is a platform role (Datatech staff) that is not tied to a tenant.

Use permissions (`services.change-price`, `invoices.discount`, `invoices.void`, `appointments.override-hours`, `giftcards.adjust`, `payroll.approve`, `journals.post`), not role names, in policies and `@can` checks. The root platform account comes from `.env` values (never hard-code a password).

---

## 5. Authentication, OTP & SMS

### Login (username, email or mobile)

```php
public function getCredentials(): array
{
    $login = trim($this->string('login')->toString());

    $field = match (true) {
        filter_var($login, FILTER_VALIDATE_EMAIL) !== false => 'email',
        preg_match('/^(\+?88)?01[3-9]\d{8}$/', $login) === 1 => 'mobile',
        default => 'username',
    };

    if ($field === 'mobile') {
        $login = normalize_bd_mobile($login); // -> 8801XXXXXXXXX, store mobiles in this format
    }

    return [$field => $login, 'password' => $this->input('password')];
}
```

Throttle login with `RateLimiter` (5 attempts per minute per login + IP).

### SmsService (transport only)

```php
declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class SmsService
{
    public function send(string $mobile, string $message): bool
    {
        $response = Http::asForm()
            ->timeout(10)
            ->post(config('services.bdbulksms.url'), [
                'to' => $mobile,
                'message' => $message,
                'token' => config('services.bdbulksms.token'),
            ]);

        // Confirm the exact success format in the BdBulkSMS API docs.
        $sent = $response->successful() && str_starts_with($response->body(), 'Ok');

        $sent
            ? Log::info('SMS sent.', ['mobile' => $mobile])
            : Log::error('SMS failed.', ['mobile' => $mobile, 'status' => $response->status()]);

        return $sent;
    }
}
```

- **Never disable SSL verification** (`'verify' => false` is forbidden).
- Config in `config/services.php`: `bdbulksms.url`, `bdbulksms.token`.
- Send SMS through a queued Job so a slow gateway never blocks a request.

### OtpService (rules)

- Generate with `random_int(100000, 999999)`.
- Store **hashed** (`Hash::make`) in Redis cache, key `otp:{mobile}`, **TTL 5 minutes**.
- Send limit: `RateLimiter` keyed by **IP + mobile** (max 5 per hour). Not session-based; clearing cookies must not bypass it.
- Verify limit: max 5 wrong attempts per code, then invalidate the code.
- Delete the code after successful verification.

---

## 6. Logging

Use `Log` with PSR-3 levels and context arrays. Never log passwords, OTPs, tokens, card or wallet numbers, or customer personal data (section 12).

```php
Log::info('Order created.', ['order_id' => $order->id, 'branch_id' => $order->branch_id]);
Log::warning('Stock below threshold.', ['ingredient_id' => $id]);
Log::error('Payment failed.', ['order_id' => $id, 'error' => $e->getMessage()]);
```

Do not log the stack trace in every message; the exception handler already does it. Log with `'exception' => $e` only when needed.

---

## 7. Activity Log & Backup

`LogsActivity` is a **trait**, not an interface:

```php
use Illuminate\Foundation\Auth\User as Authenticatable;
use Spatie\Activitylog\LogOptions;
use Spatie\Activitylog\Traits\LogsActivity;

class User extends Authenticatable
{
    use LogsActivity;

    public function getActivitylogOptions(): LogOptions
    {
        return LogOptions::defaults()
            ->logOnly(['name', 'email', 'mobile', 'username'])
            ->logOnlyDirty();
    }
}
```

- Use `logOnly([...])`, never `logAll()` (it can log password hashes).
- Explicitly log sensitive actions with `activity()->causedBy($user)->performedOn($model)->log('order.voided')`.
- Backup: `backup:run --only-db` daily via scheduler to S3 or a mounted volume, `backup:clean` weekly, and notify on failure.

---

## 8. Localization

- Files: `lang/en/*.php` and `lang/bn/*.php`. Every key must exist in both.
- A middleware sets the locale from user preference → session → default `en`.
- Vue receives translations through Inertia shared props.
- Dates and numbers: format through one helper so Bangla digits can be toggled.

---

## 9. Frontend Rules

- `<script setup>` only. One component = one responsibility.
- Show feedback on every save/delete/error with `toast.success()` / `toast.error()`.
- Handle loading, empty and error states in every list and form.
- Inertia forms use `useForm`. Show server validation errors beside fields.
- Visual and interaction rules are in `design.md`. Dialogs, confirmations, validation messages, dashboards, charts, reports and tables **must** follow sections 12 to 22 there.
- Build the shared UI kit (`resources/js/components/ui-kit/`) first and reuse it. Never use `window.alert()`, `confirm()` or `prompt()`, and never build a one-off dialog, table or chart inside a page.
- Validation lives in Form Requests. Display errors with the `FormField` component, focus the first invalid field, and never clear the form on error.

---

## 10. Testing & Definition of Done

A feature is **done** only when all of these are true:

- [ ] Pest feature tests cover the happy path and main failure cases
- [ ] Authorization tested (allowed role passes, other role gets 403, other branch gets 404, **other tenant gets 404**)
- [ ] No customer personal data in logs, URLs or error trackers (section 12)
- [ ] Where money is involved, journals balance and the reconciliation tests pass (invoices and payments, daily cash, wallet, packages, points, stock, commissions)
- [ ] `./vendor/bin/pint` is clean and `./vendor/bin/phpstan` passes
- [ ] Both `en` and `bn` translations added
- [ ] Sensitive actions write an activity log entry
- [ ] `NoEmojiTest` passes (no emojis or symbol icons in code, lang files, views or docs)
- [ ] The UI Definition of Done checklist in `design.md` (section 22) passes
- [ ] Migrations run fresh (`migrate:fresh --seed`) without errors
- [ ] The acceptance criteria for the feature in `roadmap.md` are met

---

## 11. Multi-Tenancy (SaaS)

- **Model:** one application and one shared database. Every tenant-owned table has a non-null, indexed `tenant_id`, and unique keys include it (for example `unique(tenant_id, customer_code)`).
- **Tenant resolution:** a `ResolveTenant` middleware reads the host (`{slug}.salona.example` or a custom domain). API requests take the tenant from the authenticated token. Never trust a `tenant_id` sent by the client.
- **Traits:** `BelongsToTenant` adds a global scope and fills `tenant_id` on create. `BelongsToBranch` adds branch scoping for non-owner users. An architecture test fails if a model with a `tenant_id` column lacks the trait.
- **Queues and scheduler:** every queued job, notification and listener carries the tenant ID and restores the tenant context before running. Per-tenant scheduled tasks loop over active tenants.
- **Isolation by prefix:** cache keys, Redis keys and storage paths (`tenants/{id}/...` on a private disk) all include the tenant ID.
- **Platform area (`/platform`)** for Datatech only: create tenants, plans, trial dates, subscription payments, message credits, storage and limits, custom domains, suspend or resume, usage, and support impersonation. Impersonation shows a permanent banner and is written to the audit log.
- **Plan limits** are enforced in one place, a `PlanGate` service (branches, staff, appointments per month, users, storage, modules). It returns `PLAN_LIMIT_REACHED` or `MODULE_DISABLED`. Never check plan limits in scattered controllers.
- **Tests:** every model and every endpoint has a cross-tenant test (tenant A cannot read or write tenant B, and gets 404).
- **Export and exit:** a command exports all of one tenant's data (CSV or JSON plus files). Tenant deletion is scheduled and only runs after the retention period.

---

## 12. Customer & Financial Data Rules

Salon data is personal: names, mobiles, birthdays, photos, skin and hair conditions, allergies, spending and visit patterns.

- **Never put personal data** (names, mobiles, birthdays, notes, photos, payment details) in logs, exception messages, error trackers, URLs or query strings. Use IDs only.
- **Sensitive notes** (allergies, skin conditions, medical remarks, consultation forms) and **before/after photos** are permission-protected (`customers.view-sensitive`, `photos.view`), shown only to staff who serve the customer, and stored on the private disk. Photos need recorded customer consent before they are used on the website or in marketing.
- **Encrypt** national ID, passport and bank account numbers (employees) with the `encrypted` cast. Searchable fields (name, mobile, customer code, invoice number) stay plain so they can be indexed.
- **Files** live on a private disk and are served through authorized controllers or short-lived signed URLs. Website media is copied to a separate public disk on publish.
- **Least access:** a stylist sees their own appointments and the customers they serve; receptionists see all branch customers; owners and managers see all branches they are assigned; prices, discounts, costs, commissions, payroll and profit are permission-protected (`services.view-cost`, `commission.view`, `payroll.view`, `reports.profit`).
- **Customer account area:** a customer sees only their own appointments, invoices, packages, points and gift cards. Anything else returns 404.
- **Access log:** viewing a customer's sensitive notes or photos, and exporting customers, writes an access log row. Bulk exports need a permission and are logged.
- **No hard deletes** of customers with history, invoices, payments, wallet or points entries, journals or employees with payroll. Use statuses or voiding with a reason. Anonymize a customer on request only through a documented command that keeps financial totals.
- **Consent and opt-out:** marketing messages go only to customers who have not opted out; transactional messages (confirmation, reminder, receipt) are always allowed. The opt-out is honoured across SMS, WhatsApp and email.
- **Sessions:** idle timeout (default 30 minutes in the admin), strong password policy, optional TOTP for owners and accountants.
- **Backups** are encrypted and a restore drill is documented. Confirm data protection and consumer rules (gift card validity, membership refunds, VAT) with a legal advisor and accountant before launch.

---

## 13. Public Website, SEO & Online Booking

- **One codebase, many websites.** A `ResolveWebsite` middleware maps the host to a tenant: `{slug}.salona.example` or a verified custom domain with automatic TLS through a front proxy. Unknown hosts return 404, never another tenant's site.
- **Rendering:** Inertia SSR with dynamic meta, OpenGraph and Twitter tags. Public pages are cached per tenant and language (Redis tags, `tenant:{id}`) and purged on publish or when services, prices, offers or staff change. Target: LCP under 2.5 seconds on a mid-range phone, Lighthouse SEO and accessibility 95 or more. The booking widget is the only interactive part and loads lazily.
- **URLs and languages:** clean slugs, `/en/...` and `/bn/...` with `hreflang`, canonical URLs, a language switcher that keeps the page. A missing translation falls back to the default language and is flagged to editors.
- **SEO:** per-page meta fields, sitemap per tenant, `robots.txt`, redirects manager, and JSON-LD: `HealthAndBeautyBusiness` with opening hours and branch address, `Service`, `Offer`, `BreadcrumbList`, `Article`, `JobPosting`.
- **Media pipeline:** uploads validated and processed in a queue into WebP derivatives with responsive sizes; originals stay private; alt text required; gallery images need customer consent when they show a customer; video is an external embed.
- **Online booking** calls the same `BookAppointmentAction` as the admin calendar. The public flow is: branch, services (or package), preferred staff or "any", date and time from the slot engine (section 14), customer name and mobile, OTP verification (to stop fake bookings), confirmation. It creates or merges the customer by mobile and writes the appointment, so **every online booking enters the CRM automatically**. It is idempotent per submission token, uses Turnstile and rate limits, and never reveals whether a mobile already exists.
- **Deposits** (optional per service or tenant) are recorded as a payment on the appointment; online gateway payment is out of scope in v1, so deposits are taken in person or by mobile banking with verification.
- **Pricing pages** show only published prices; per-branch price differences are respected; offers and memberships show validity dates.
- **Forms:** contact, careers and membership enquiries post to one `CaptureLeadAction` (anti-spam, merge by mobile, notify). Privacy and terms pages are required before publishing a form.

---

## 14. Appointment & Scheduling Integrity

- **Slot engine:** one `AvailabilityService` computes bookable slots from branch opening hours and holidays, staff shifts and leave, staff skills (which services a person can perform), service duration plus buffer, resource needs (chair, bed, room) and existing appointments. The admin calendar, online booking and reschedule all use it. No second implementation.
- **No double booking:** creating or moving an appointment locks the calendar rows of the staff member and required resources (`lockForUpdate()`) inside a transaction, re-checks the slot, and writes the appointment. A multi-service appointment is a set of **appointment items**, each with its own staff, resource, start and end, so two stylists can work in sequence or in parallel. Concurrency tests are mandatory.
- **State machine** changed only through `ChangeAppointmentStatusAction`: `booked -> confirmed -> checked_in -> in_service -> completed`, plus `cancelled`, `no_show` and `rescheduled` (a reschedule keeps the old times in history). Illegal transitions are rejected and every change writes a history row.
- **Walk-ins** use the same action and the next free slot or "now", with the queue shown on the reception screen.
- **Cancellation and no-show:** rules per tenant (free cancellation window, fee, deposit forfeiture). No-shows are counted per customer; a threshold can require a deposit or block online booking, always overridable by staff with a reason.
- **Reminders:** idempotent jobs send confirmation, reminder (for example 24 hours and 2 hours before) and post-visit thank-you, using `MessageChannel`. A cancelled or moved appointment cancels pending reminders.
- **Waitlist:** a cancelled slot can notify customers on the waitlist; the first confirmed reply wins under a lock.
- **Staff overrides:** booking outside hours or over a break needs `appointments.override-hours` and is logged.
- **Calendar views** (day, week, by staff, by resource, by branch) read from the same tables; drag and drop calls the same actions.

---

## 15. Billing, Loyalty, Membership & Accounting Integrity

- **Invoice from appointment:** completing an appointment creates a draft invoice from its items; the cashier can add retail products, discounts, coupons, a gift card, wallet balance, tips, and then finalizes. A finalized invoice is **immutable**; corrections use a credit note or a void with a reason and approval. Invoice numbers are sequential per branch series under a lock, without gaps; voided numbers stay visible.
- **Split payments** (cash, card or POS, mobile banking with TrxID, bank transfer, wallet, gift card, package credit) are stored as payment rows that must sum to the invoice total. **Customer dues** (partial payment) are explicit receivables with their own collection receipts.
- **Ledgers, not editable balances.** Wallet (prepaid) balance, **package sessions remaining**, **loyalty points**, **gift card balance** and referral rewards are each an append-only ledger. A balance is the sum of entries and is never typed. Every entry references the invoice or adjustment that caused it. Reconciliation tests are mandatory.
- **Packages and memberships:** selling a package creates a package instance with sessions or value, expiry and eligible services; using a session writes a ledger entry and links the staff member who performed it. A membership plan applies benefits (discount percent, free services, extra points, priority) while active; expiry and renewal are handled by an idempotent job. Refunds of packages follow the tenant rule (unused value minus deduction) through approval.
- **Loyalty points:** earned on finalized invoices by rule (per BDT spent, per service), redeemed as payment, expire by rule, and reverse when the invoice is voided or credited. **Coupons** have validity, usage limits, branch and service scope, and single-use enforcement under a lock. **Gift cards** have a code, value, expiry and balance ledger; codes are random and not guessable.
- **Referral rewards** are granted once, when the referred customer completes a first paid invoice, through the same points or wallet ledger.
- **Commission and incentives:** calculated per invoice item when the invoice is finalized, from the service's commission rule (percentage or fixed, by staff level or branch) and stored as commission entries. A void or credit note reverses them. Monthly statements are approved and locked before payroll uses them. Sales commission on retail products is a separate rule.
- **Accounts:** all postings go through `PostJournalEntryAction`; journals balance; posted entries are immutable; periods can be locked. The business event and its posting happen in one transaction. Account mapping (sales by category, tips payable, wallet liability, gift card liability, package deferred revenue, cash, bank, expense categories) is configuration. Prepaid wallet, packages and gift cards are liabilities until used.
- **Daily cash:** each branch closes the day by counting cash against the system total; shortage or excess is recorded with approval, never hidden. A closed day rejects back-dated edits.
- **Supplier and employee payments** are vouchers with approval above a limit.
- Dashboards must equal reports and ledgers; add reconciliation tests for sales, payments, daily cash, wallet, packages, points, gift cards, stock value, commissions and profit.

---

## 16. Inventory & Consumption

- **Stock changes only through immutable stock movements** posted by `PostStockMovementAction` under a row lock: purchase receipt, transfer in or out, retail sale, service usage, wastage, adjustment, stock count. The balance equals the sum of movements per product and branch. Negative stock is blocked unless the tenant allows it.
- **Product types:** retail products (sold to customers), consumables (used in services, measured in a base unit such as ml or piece), and equipment (not stock-tracked).
- **Service recipes:** a service can list the products and quantities it normally uses. Finalizing the invoice (or completing the service) posts usage movements automatically; staff may record the actual amount when it differs, and the variance is reported as wastage or overuse.
- **Costing:** weighted average cost per product and branch, recalculated on receipt; the cost used is stored on the movement. Cost data needs `services.view-cost`.
- **Purchases:** suppliers, purchase orders, goods receipt, returns; supplier bills create payables.
- **Alerts:** reorder levels per product and branch with an idempotent daily low-stock job; expiry dates optional per batch with near-expiry alerts.
- **Transfers** between branches are two-step (send, receive) with differences recorded.
- **Stock count** with variance review and approval before adjustments post.

---

## 17. Marketing, Messaging, Tasks & Scheduled Jobs

- **Segmentation:** customers are classified by rules computed by an idempotent job and stored with a timestamp: **inactive** (no completed visit in N days, default 60), **VIP** (spend or visits over thresholds), **new**, **birthday this week**, **anniversary this week**, **membership expiring**, **package expiring**, and custom filters. A segment is a saved filter; counts are exact at send time.
- **Campaigns:** choose a segment, a template (SMS, WhatsApp, email), an optional coupon, and a send time. The screen shows audience size, message segments and cost before sending; opted-out customers are excluded; a send is idempotent per campaign and customer. Automatic journeys (birthday wish, "we miss you" at 60 days, post-visit thank you and review request, membership renewal) are configured rules, not code per campaign. A campaign records messages sent, bookings and revenue attributed within a window.
- **Messaging:** a `MessageChannel` interface with SMS (BdBulkSMS), WhatsApp Cloud API (approved templates) and email. Templates per tenant, channel and language with placeholders and preview. Sends are queued, logged, rate limited, and consume prepaid credits in the same transaction that writes the log. Bangla SMS is Unicode (70 characters per segment); always show the segment count and cost before a bulk send.
- **Tasks:** a generic `tasks` table (assignee, due date, status, related entity), used for follow-ups, stock tasks and manager to-dos.
- **Approvals:** one approval engine for discounts above a limit, voids, credit notes, refunds, price changes, expenses, leave, payroll, commission payout, stock adjustments and unlocking a day. Do not write special approval code per feature.
- **Imports:** `maatwebsite/excel`, queued, with dry run, row errors, safe re-run keyed on external ID; covers customers (with visit history summary and wallet or package balances), services, products with opening stock, staff.
- **Scheduled jobs** (reminders, segment refresh, journeys, package and membership expiry, point expiry, low-stock alerts, daily owner summary, day-close reminder, trial expiry, backups) must be **idempotent** and tenant-aware: running twice never duplicates anything, and one failing tenant never stops the others.

---

## 18. Multi-Branch & Reports

- **Branches** belong to a tenant (a head office plus branches, or a chain). Users have a branch scope; owners and head-office roles see all. `BelongsToBranch` enforces scope; reports have a branch filter and a consolidated view.
- **Shared and per-branch settings:** services and products are defined once at tenant level; price, availability, duration override, stock and staff are set per branch. Customers are tenant-wide so a chain recognizes a customer at any branch; each visit records its branch. Wallet, packages, points and gift cards are redeemable across branches unless the tenant restricts them, and inter-branch settlement is reported.
- **Dashboards** read from aggregated tables refreshed by queued jobs plus live queries for today's numbers. Every KPI links to a report with the same filters and the same total.
- **Fixed KPI definitions** are documented in `prd.md` section 5 (revenue, appointments, utilization, no-show rate, retention, average ticket) so every screen agrees.
- Long reports (more than about 10,000 rows) run as queued jobs with a notification and download link.

---

## 19. Working Method (for the AI)

1. Read `docs/prd.md`, `docs/api.md`, `docs/design.md`, `docs/roadmap.md`.
2. Build **one phase at a time** from `roadmap.md`. Do not start the next phase.
3. For each feature: migration, model, Form Request, Action, controller, Vue page, tests.
4. If something in the docs is unclear or contradictory, **ask before assuming**.
5. Run tests and fix failures before reporting a feature as finished.
