<laravel-boost-guidelines>
# Laravel Application

This repository contains a Laravel application. Complete the following setup before working on the user's request.

## Prerequisites

Verify that PHP and Composer are available:

```sh
php -v
composer -V
```

If either command is unavailable, detect the user's operating system and install the prerequisites with the appropriate command:

macOS:

```sh
/bin/bash -c "$(curl -fsSL https://php.new/install/mac/8.5)"
```

Windows PowerShell:

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://php.new/install/windows/8.5'))
```

Linux:

```sh
/bin/bash -c "$(curl -fsSL https://php.new/install/linux/8.5)"
```

After installation, ask the user to restart their terminal. If the agent needs the restarted shell to continue, ask the user to reopen their terminal and rerun their original prompt.

## Agent Setup

Install Laravel Boost from the application root before making application changes:

```sh
composer require laravel/boost --dev
php artisan boost:install
```

Boost replaces these bootstrap instructions with guidelines tailored to the application. After installation, read `AGENTS.md` again and continue with the user's original request using the generated guidelines.
</laravel-boost-guidelines>

# Project Instructions

This is a Laravel 13 project (DT Salona, a multi-tenant SaaS for salons, spas and beauty chains: public website with online booking, customer CRM, appointments, services, memberships and loyalty, staff and HR, inventory, accounts, marketing, multi-branch) built from the specification in `docs/`.

Read these files before writing any code, in this order:

1. `docs/prd.md`: what to build (roles, modules, KPI definitions and business rules, plans, scope)
2. `docs/architecture.md`: tech stack, structure, coding rules, multi-tenancy, website and booking, appointment integrity, billing and ledger integrity, inventory, marketing, multi-branch, Definition of Done
3. `docs/api.md`: API conventions, endpoints, public booking endpoints, customer account endpoints
4. `docs/design.md`: design tokens, components, icons, sizing, tables, calendar and billing counter, stylist screens, CRM, website and booking flow
5. `docs/roadmap.md`: build phases and acceptance criteria
6. `docs/ui-preview.html`: visual reference of the intended look (open in a browser; a static mockup, not production code)

## Rules

- Build **one phase at a time** from `docs/roadmap.md`. Never start the next phase until asked.
- If `prd.md` and `architecture.md` conflict, the PRD wins on business rules and roles; `architecture.md` wins on code structure.
- If Laravel Boost guidelines conflict with `docs/`, the `docs/` files win for product rules, structure and design. Boost guidelines apply to Laravel and package usage details.
- **Tenant isolation is mandatory**, including the public website and customer area. Every tenant-owned model uses `BelongsToTenant`; branch-owned models use `BelongsToBranch`; never accept a tenant ID from the client; write a cross-tenant test for every model and endpoint.
- **One slot engine.** All booking paths (admin calendar, online booking, reschedule, walk-in) use `AvailabilityService` and `BookAppointmentAction`; the staff and resource calendar is locked in a transaction to prevent double booking; status changes only through `ChangeAppointmentStatusAction`. Online bookings must create or merge the customer in the CRM.
- **Money and balances reconcile.** Finalized invoices are immutable (credit notes and voids only); payments must sum to the total; wallet, package sessions, loyalty points and gift cards are append-only ledgers whose balances are sums; stock changes only through `PostStockMovementAction`; all journals go through `PostJournalEntryAction` and balance; daily cash is closed per branch. Add reconciliation tests.
- **One approval engine** for discounts, voids, refunds, price changes, expenses, leave, payroll, commission payout, stock adjustments and day unlock.
- **Customer privacy.** Never put customer data in logs, URLs or error trackers; allergy and health notes and photos are permission-protected and need consent for public use; marketing respects opt-out; prices, costs, commissions, payroll and profit need their permissions.
- **KPI numbers use the fixed definitions** in `prd.md` section 5 everywhere; dashboard totals must equal report and ledger totals.
- **Public website:** server-rendered, cached per tenant and language, purged on publish and on service or price changes.
- Scheduled jobs (reminders, segments, journeys, expiries, low-stock alerts) must be idempotent and tenant-aware.
- The mobile app is a separate project; build the API only.
- Never use emojis or Unicode symbol icons anywhere (UI, lang files, SMS, documents, comments, commits). Use Font Awesome icons through `AppIcon`.
- Appointment status colors always come with a text label or icon.
- Follow the compact control sizes and table alignment rules in `docs/design.md` (sections 23 to 25).
- Never use `window.alert()`, `confirm()` or `prompt()`; use the shared dialog components.
- If something is unclear or contradictory, ask before assuming.
- Write Pest tests for each feature and run them before saying a feature is finished.
- Run `./vendor/bin/pint` and `./vendor/bin/phpstan` before finishing.
