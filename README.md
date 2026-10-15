# DT Salona

**From appointment to payment, customer to employee: manage your entire salon from one platform.**
Developed by **Datatech BD Ltd**, Dhaka.

DT Salona gives salons, spas and beauty chains a professional website with online booking, and runs the calendar, billing counter, customer CRM, memberships and loyalty, staff and commission, inventory, accounts and marketing behind it, across one branch or many.

```text
Website / Phone / Walk-in -> Appointment -> Service -> Billing -> Payment -> Loyalty / Package
```

Built for beauty salons, hair salons, spas, beauty parlours, makeup studios, men's grooming and multi-branch chains. Bilingual (English and Bangla). Multi-tenant SaaS: many businesses on one platform.

> This project is built phase by phase from the specification in `docs/`. Start with `docs/roadmap.md`. The mobile app is a separate project; this repository provides the API.

---

## Features

- **Public website and online booking:** services, pricing, packages, team, gallery, offers, membership, blog, careers, SEO, custom domain; bookings land in the calendar and CRM automatically
- **Customer CRM:** profiles, visit and service history, preferences, birthdays, VIP and inactive segments
- **Appointments:** calendar by staff, resource and branch, walk-in queue, reschedule, cancellation, no-show tracking, reminders, waitlist
- **Services:** price, duration, staff commission, required products, branch availability, packages
- **Membership and loyalty:** plans, prepaid packages, points, coupons, gift cards, wallet, referral rewards
- **Billing:** split payments, tips, dues, void and credit notes, daily cash close
- **Staff and HR:** attendance, shifts, leave, commission, incentives, performance, payroll
- **Inventory:** retail and consumable stock, purchases, suppliers, usage by recipe, wastage, low-stock alerts, branch transfers
- **Accounts:** sales, expenses, supplier and staff payments, customer dues, bank, profit and loss by branch
- **Marketing:** win-back campaigns ("We miss you! Get 15% off your next visit."), birthday automations, SMS, WhatsApp and email
- **Multi-branch:** head office and branches with consolidated management view
- **Platform console (Datatech):** tenants, plans, trials, credits, domains, import monitor

Full scope and business rules: `docs/prd.md`.

---

## Tech Stack

| Area                  | Technology                                                              |
| --------------------- | ----------------------------------------------------------------------- |
| Backend               | Laravel 13.x, PHP 8.3+                                                  |
| Database              | MySQL 8 (shared database, `tenant_id` isolation)                        |
| Cache, session, queue | Redis                                                                   |
| Frontend              | Inertia.js v2, Vue 3, Tailwind CSS v4, shadcn-vue, SSR for public pages |
| Icons, charts, maps   | Font Awesome Free, Chart.js, Leaflet                                    |
| Auth and roles        | Laravel Sanctum, Spatie Permission (teams), customer OTP guard          |
| Audit and backup      | Spatie Activitylog, Spatie Backup                                       |
| PDF, import, media    | spatie/laravel-pdf (Browsershot), Laravel Excel, Intervention Image     |
| Messaging             | BdBulkSMS, Laravel Mail, WhatsApp Cloud API                             |
| SEO                   | spatie/laravel-sitemap, JSON-LD, hreflang                               |
| Testing and quality   | Pest, Playwright, Laravel Pint, Larastan                                |
| Containers            | Docker and Docker Compose                                               |

---

## Documentation

AI coding agents read `CLAUDE.md` first.

| File                   | Purpose                                                                                                                           |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `docs/prd.md`          | Product requirements: roles, modules, KPI definitions, business rules, plans, scope                                               |
| `docs/architecture.md` | Stack, structure, coding rules, multi-tenancy, website and booking, appointment, billing and ledger integrity, Definition of Done |
| `docs/api.md`          | API conventions, public booking, back-office, customer and platform endpoints                                                     |
| `docs/design.md`       | Design system, calendar and billing counter, stylist screens, CRM, website and booking flow, icons, sizing, tables                |
| `docs/roadmap.md`      | Build phases with acceptance criteria                                                                                             |
| `docs/ui-preview.html` | Static visual mockup of the intended UI (open in a browser)                                                                       |

---

## Getting Started

### Requirements

- Docker and Docker Compose, Git
- Node.js 20+ and npm (only if you run the frontend outside Docker)

### Setup

```sh
git clone <repository-url> dt-salona
cd dt-salona

cp .env.example .env
# Edit .env: database, Redis, mail, BdBulkSMS, WhatsApp, Turnstile, map tiles

docker compose up -d --build
docker compose exec app composer install
docker compose exec app php artisan key:generate
docker compose exec app php artisan migrate --seed
docker compose exec app npm install
docker compose exec app npm run dev
```

The service name `app` follows the Docker Compose file created in Phase 0; adjust if yours differs.

The platform administrator is created by the seeder from `.env` (never commit real values):

```text
PLATFORM_ADMIN_NAME=
PLATFORM_ADMIN_EMAIL=
PLATFORM_ADMIN_PASSWORD=
```

For local development the seeder also creates two demo salon businesses so tenant isolation can be tested. Demo websites are served on `{slug}.salona.test` hosts (add them to your hosts file or use a local wildcard DNS).

### Background processes

| Process                                                                                         | Command                     |
| ----------------------------------------------------------------------------------------------- | --------------------------- |
| Queue worker                                                                                    | `php artisan queue:work`    |
| Scheduler (reminders, segments, automations, expiries, low-stock alerts, trial expiry, backups) | `php artisan schedule:work` |

---

## Development Workflow

```sh
php artisan test            # or ./vendor/bin/pest
./vendor/bin/pint           # code style
./vendor/bin/phpstan        # static analysis
php artisan migrate:fresh --seed
```

Rules:

- Build one roadmap phase at a time; finish its acceptance criteria and the Definition of Done in `docs/architecture.md`.
- Every feature has Pest tests, including authorization, scope, and **cross-tenant isolation**.
- Appointments only through `BookAppointmentAction` and `ChangeAppointmentStatusAction`; ledgers append-only; journals only through `PostJournalEntryAction`; reconciliation tests must pass.
- Never put customer personal data in logs or URLs.
- No emojis anywhere. Icons come from Font Awesome through `AppIcon`.
- Every text key exists in both `lang/en` and `lang/bn`.

---

## Roles

Tenant roles: `owner`, `branch-manager`, `receptionist`, `stylist`, `cashier`, `inventory-manager`, `accountant`, `hr-manager`, `marketing-manager`, `web-editor`. Customers log in to their account area with mobile OTP. Platform role: `super-admin`.

---

## Deployment Notes

- HTTPS only; `APP_ENV=production`, `APP_DEBUG=false`.
- Custom domains need a front proxy with automatic TLS (for example Caddy on-demand TLS or a CDN service) and DNS verification.
- Run the queue worker and scheduler under a process supervisor.
- The PHP image must include Chromium and the Inter and Noto Sans Bengali fonts for PDFs.
- Schedule encrypted daily backups and test a restore before onboarding real companies.
- Run the nightly reconciliation (invoices and payments, daily cash, ledgers, stock, profit) and alert on any difference.
- Confirm VAT, gift card and membership terms, refund rules, data protection, and message sender ID rules with a lawyer and accountant.

---

## Support

Datatech BD Ltd, Dhaka, Bangladesh.

## License

Proprietary. All rights reserved by Datatech BD Ltd. Not for redistribution.
