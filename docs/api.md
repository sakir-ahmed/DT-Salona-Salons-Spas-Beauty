# API SPECIFICATION: DT Salona

> Sections 1 and 2 are shared conventions. Sections 3 onward are specific to DT Salona.

---

## 1. Purpose & Conventions

**Who uses the API:** the public website (online booking and enquiries), the customer account area, a future mobile app (separate project), and integrations. **The admin web app uses Inertia controllers (session auth)** that call the _same Actions_ as the API. Do not duplicate business logic.

| Topic         | Rule                                                                                                                                                                                            |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Base path     | `/api/v1/...` in `routes/api.php`                                                                                                                                                               |
| Tenant        | Taken from the authenticated token, or from the website host for `/api/v1/public`. **Never accepted from the client body.**                                                                     |
| Auth          | Laravel Sanctum bearer tokens. Token abilities: `staff`, `customer`. Public endpoints need no token.                                                                                            |
| Docs          | OpenAPI generated automatically (Scramble) at `/api-docs`; every endpoint has request and response examples.                                                                                    |
| Versioning    | No breaking changes inside `v1`: add fields freely, never rename or remove. Clients send `X-App-Version` and `X-Platform`.                                                                      |
| Format        | JSON. Success: `{"success": true, "data": {...}, "message": "Localized message"}`. Error: `{"success": false, "error": {"code": "SLOT_NOT_AVAILABLE", "message": "...", "details": {...}}}`     |
| Language      | `Accept-Language: en` or `bn`                                                                                                                                                                   |
| Money         | Decimal **string** in BDT, e.g. `"4800.00"`                                                                                                                                                     |
| Duration      | Whole minutes (integer)                                                                                                                                                                         |
| Dates         | ISO 8601 with timezone (`Asia/Dhaka`)                                                                                                                                                           |
| Pagination    | `?page=1&per_page=25`; response includes `meta` (`current_page`, `last_page`, `total`)                                                                                                          |
| Filtering     | `?filter[status]=confirmed&filter[branch]=12&sort=-created_at`                                                                                                                                  |
| Idempotency   | `POST` of public bookings, invoices, payments, refunds, vouchers, journals, payroll runs, imports and bulk messages requires an `Idempotency-Key` header (public forms use a submission token). |
| Rate limits   | Login, OTP and public forms: strict. Others: 120 per minute per token.                                                                                                                          |
| Scope         | Stylists see only their own appointments; branch users see only their branches; customers see only their own data. Every branch route checks access, else **404**.                              |
| Personal data | No customer data in URLs or query strings (use IDs).                                                                                                                                            |

### Error codes

| HTTP | Code                                                                     | When                                                                                                  |
| ---- | ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| 401  | `UNAUTHENTICATED`                                                        | Missing or invalid token                                                                              |
| 403  | `FORBIDDEN`                                                              | Role lacks permission                                                                                 |
| 404  | `NOT_FOUND`                                                              | Missing, or belongs to another scope, user or tenant                                                  |
| 422  | `VALIDATION_FAILED`                                                      | Invalid input, `details` has field errors                                                             |
| 409  | `SLOT_NOT_AVAILABLE`                                                     | Staff or resource already booked, outside hours, or on leave (`details` has the next available slots) |
| 409  | `ILLEGAL_APPOINTMENT_TRANSITION`                                         | Status change not allowed                                                                             |
| 409  | `CANCELLATION_WINDOW_CLOSED`                                             | Customer cancel or reschedule after the allowed time                                                  |
| 402  | `DEPOSIT_REQUIRED`                                                       | The customer must pay a deposit (no-show threshold or service rule)                                   |
| 422  | `OTP_INVALID`                                                            | Wrong or expired OTP                                                                                  |
| 422  | `PAYMENT_TOTAL_MISMATCH`                                                 | Payments do not equal the invoice total                                                               |
| 422  | `DISCOUNT_NOT_ALLOWED`                                                   | Discount over the role's limit and no approval path                                                   |
| 202  | `APPROVAL_REQUIRED`                                                      | Request created and waiting for approvers (`details` has approval id)                                 |
| 409  | `INVOICE_ALREADY_FINALIZED`                                              | Editing a finalized invoice (use a credit note or void)                                               |
| 409  | `INVOICE_ALREADY_VOIDED`                                                 | Voiding twice                                                                                         |
| 409  | `DAY_CLOSED`                                                             | Posting into a closed cash day                                                                        |
| 422  | `COUPON_INVALID`                                                         | Expired, used up, wrong scope or already redeemed                                                     |
| 422  | `GIFT_CARD_INSUFFICIENT` / `WALLET_INSUFFICIENT` / `POINTS_INSUFFICIENT` | Balance too low                                                                                       |
| 422  | `PACKAGE_EXHAUSTED` / `PACKAGE_EXPIRED`                                  | No sessions left or expired                                                                           |
| 409  | `INSUFFICIENT_STOCK`                                                     | Not enough stock in the branch (`details` has available)                                              |
| 422  | `JOURNAL_UNBALANCED`                                                     | Debits and credits differ                                                                             |
| 409  | `PERIOD_LOCKED`                                                          | Posting into a locked period                                                                          |
| 409  | `COMMISSION_STATEMENT_LOCKED`                                            | Editing an approved commission statement                                                              |
| 403  | `PRICE_NOT_ALLOWED` / `COST_NOT_ALLOWED` / `SENSITIVE_NOT_ALLOWED`       | Reading prices, costs or sensitive customer data without permission                                   |
| 403  | `PLAN_LIMIT_REACHED` / `MODULE_DISABLED`                                 | Branch, staff, user or module limit                                                                   |
| 403  | `TENANT_SUSPENDED` / `TENANT_READ_ONLY`                                  | Trial ended or account suspended                                                                      |
| 402  | `MESSAGE_CREDIT_INSUFFICIENT`                                            | Not enough SMS or WhatsApp credits                                                                    |
| 429  | `TOO_MANY_REQUESTS`                                                      | Rate limit hit                                                                                        |
| 500  | `SERVER_ERROR`                                                           | Unexpected (details never exposed)                                                                    |

---

## 2. Authentication

| Method | Path                                      | Notes                                                                              |
| ------ | ----------------------------------------- | ---------------------------------------------------------------------------------- |
| POST   | `/api/v1/auth/login`                      | Staff `login` (username, email or mobile) + `password`; returns token.             |
| POST   | `/api/v1/auth/logout`                     | Revokes the current token                                                          |
| GET    | `/api/v1/auth/me`                         | User, role, permissions, branches, plan limits, trial status, feature flags        |
| POST   | `/api/v1/auth/otp/send` and `/otp/verify` | Customers; rate limited by IP and mobile; hashed code, 5-minute expiry, 5 attempts |

---

## 3. Public Website & Booking Endpoints (`/api/v1/public`, tenant from host)

| Method | Path                                                                         | Notes                                                                                                         |
| ------ | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| GET    | `/site`                                                                      | Branding, menus, languages, branches with hours, contact data                                                 |
| GET    | `/services`, `/services/{slug}`, `/packages`, `/membership-plans`, `/offers` | Publishable data only, branch-specific prices                                                                 |
| GET    | `/team`                                                                      | Staff shown on the website                                                                                    |
| GET    | `/pages/{slug}`, `/posts`, `/posts/{slug}`, `/jobs`                          | Bilingual content with fallback                                                                               |
| GET    | `/booking/slots`                                                             | Query: `branch`, `services[]`, optional `staff`, `date`; returns bookable slots from `AvailabilityService`    |
| POST   | `/booking/otp`                                                               | Send OTP to the mobile (rate limited)                                                                         |
| POST   | `/booking/appointments`                                                      | Verified OTP, submission token, Turnstile; creates or merges the customer and the appointment; generic errors |
| POST   | `/booking/waitlist`                                                          | Join the waitlist for a full day                                                                              |
| POST   | `/enquiries`                                                                 | Contact, membership enquiry, callback; goes to `CaptureLeadAction`                                            |
| POST   | `/job-applications`                                                          | Applicant with CV (private file)                                                                              |

---

## 4. Back-Office Endpoints

### Setup & Catalog

| Method       | Path                                                               | Notes                                                                                          |
| ------------ | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| GET/POST/PUT | `/branches`, `/branches/{branch}/hours`, `/holidays`, `/resources` | Opening hours, closures, chairs and rooms                                                      |
| GET/POST/PUT | `/service-categories`, `/services`, `/service-packages`            | Recipe, commission rule and price fields need `services.view-cost` and `services.change-price` |
| GET/POST/PUT | `/services/{service}/branch-settings`                              | Per-branch price, duration override, availability                                              |
| GET/POST/PUT | `/price-lists`                                                     | Effective dates                                                                                |
| POST         | `/imports/{type}`                                                  | `customers`, `services`, `products`, `staff`; `dry_run` then commit                            |

### Customers & CRM

| Method       | Path                                                  | Notes                                                                                |
| ------------ | ----------------------------------------------------- | ------------------------------------------------------------------------------------ |
| GET/POST/PUT | `/customers`, `/customers/{customer}`                 | Duplicate detection by mobile; sensitive fields only with `customers.view-sensitive` |
| POST         | `/customers/merge`                                    | Manual merge with history                                                            |
| GET/POST     | `/customers/{customer}/notes`, `/photos`, `/consents` | Permission and access log                                                            |
| GET          | `/customers/{customer}/history`                       | Visits, services, invoices                                                           |
| GET/POST/PUT | `/segments`                                           | Saved rules; `GET /segments/{segment}/customers` with exact count                    |
| GET/POST     | `/customers/{customer}/consultation-forms`            | Signed forms                                                                         |

### Appointments

| Method   | Path                                     | Notes                                                                                        |
| -------- | ---------------------------------------- | -------------------------------------------------------------------------------------------- |
| GET      | `/calendar?view=day&branch=&date=`       | Appointment items grouped by staff or resource                                               |
| GET      | `/availability/slots`                    | Same engine as the public endpoint, with staff-only options (override hours)                 |
| POST     | `/appointments`                          | `BookAppointmentAction`; multiple items with own staff and start; `Idempotency-Key` required |
| PUT      | `/appointments/{appointment}`            | Move, change staff or services                                                               |
| POST     | `/appointments/{appointment}/status`     | `confirm`, `check-in`, `start`, `complete`, `cancel` (reason), `no-show`                     |
| POST     | `/appointments/{appointment}/reschedule` | Keeps history                                                                                |
| POST     | `/walk-ins`                              | Add to the queue or next free slot                                                           |
| GET/POST | `/waitlist`                              | List and offer freed slots                                                                   |

### Billing & Payments

| Method   | Path                                           | Notes                                                                 |
| -------- | ---------------------------------------------- | --------------------------------------------------------------------- |
| POST     | `/invoices`                                    | Draft from an appointment or a walk-in                                |
| PUT      | `/invoices/{invoice}`                          | Edit draft lines, discounts, tips, staff credit                       |
| POST     | `/invoices/{invoice}/finalize`                 | Payments array that must sum to the total; `Idempotency-Key` required |
| POST     | `/invoices/{invoice}/void` and `/credit-notes` | Reason and approval                                                   |
| GET      | `/invoices/{invoice}`                          | JSON or PDF                                                           |
| GET/POST | `/dues` and `/dues/{due}/payments`             | Customer dues and collection receipts                                 |
| POST     | `/coupons/validate` and `/gift-cards/validate` | Check before applying                                                 |
| POST     | `/cash-days/{day}/close`                       | Counted cash; differences go to approval                              |
| POST     | `/cash-days/{day}/unlock`                      | Approval required                                                     |

### Membership & Loyalty

| Method       | Path                                                           | Notes                                                                     |
| ------------ | -------------------------------------------------------------- | ------------------------------------------------------------------------- |
| GET/POST/PUT | `/membership-plans`, `/memberships`                            | Sell, renew, cancel                                                       |
| GET/POST     | `/packages`, `/customer-packages`                              | Sell prepaid sessions; redemption happens on invoices                     |
| GET          | `/customers/{customer}/ledgers/{type}`                         | `wallet`, `points`, `package`, `gift-card` ledger entries                 |
| POST         | `/wallet/top-ups`                                              | Adds ledger entry and payment                                             |
| POST         | `/adjustments`                                                 | Manual adjustment of wallet, points or gift card with reason and approval |
| GET/POST/PUT | `/coupons`, `/gift-cards`, `/loyalty-rules`, `/referral-rules` | Configuration                                                             |

### Staff / HR

| Method       | Path                                                                 | Notes                                                             |
| ------------ | -------------------------------------------------------------------- | ----------------------------------------------------------------- |
| GET/POST/PUT | `/employees`, `/shifts`, `/rosters`, `/leave-types`                  | Payroll data needs `payroll.view`                                 |
| GET/POST     | `/attendance`, `/attendance/import`                                  | Web check-in, manual, import                                      |
| GET/POST     | `/leave-requests`                                                    | Approval engine                                                   |
| GET/POST/PUT | `/commission-rules`, `/incentive-rules`                              | `commission.view`                                                 |
| GET          | `/commission-statements`; POST `/commission-statements/{id}/approve` | Approval locks the statement                                      |
| POST         | `/payroll/runs`; POST `/payroll/runs/{run}/approve` and `/post`      | Locks the month and posts the journal; `Idempotency-Key` required |
| GET          | `/staff-performance`                                                 | Services, revenue, rebooking, utilization                         |
| GET/POST     | `/job-posts`, `/applicants`                                          | Recruitment                                                       |

### Inventory

| Method       | Path                                                             | Notes                                 |
| ------------ | ---------------------------------------------------------------- | ------------------------------------- |
| GET/POST/PUT | `/products`, `/suppliers`, `/purchase-orders`, `/goods-receipts` | Cost fields need `services.view-cost` |
| GET          | `/stock?branch=` and `/stock/movements`                          | Balances and ledger                   |
| POST         | `/stock/transfers` and `/transfers/{transfer}/receive`           | Two-step with differences             |
| POST         | `/stock/wastage` and `/stock/adjustments`                        | Reason and approval above limit       |
| POST         | `/stock/counts` and `/counts/{count}/approve`                    | Variance review                       |
| GET/PUT      | `/services/{service}/recipe`                                     | Products and quantities per service   |

### Accounts

| Method       | Path                                                                                            | Notes                                                                                               |
| ------------ | ----------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| GET/POST/PUT | `/accounts`, `/fiscal-years`, `/periods`                                                        | Lock and unlock a period (approval)                                                                 |
| POST         | `/vouchers`                                                                                     | Receipt, payment, journal, contra; `PostJournalEntryAction`                                         |
| POST         | `/vouchers/{voucher}/reverse`                                                                   | Reason and approval                                                                                 |
| GET/POST     | `/expenses`, `/supplier-bills`, `/supplier-payments`, `/bank-accounts`, `/bank-reconciliations` | Approval above limit                                                                                |
| GET          | `/reports/{report}`                                                                             | `profit-loss`, `branch-profit`, `cash-flow`, `receivables`, `payables`, `trial-balance`, `day-book` |

### Marketing & Messaging

| Method       | Path                                          | Notes                                    |
| ------------ | --------------------------------------------- | ---------------------------------------- |
| GET/POST/PUT | `/message-templates`                          | Channel, language, placeholders, preview |
| POST         | `/campaigns/estimate`                         | Audience size, segments, cost            |
| POST         | `/campaigns` and `/campaigns/{campaign}/send` | `Idempotency-Key` required               |
| GET/POST/PUT | `/automations`                                | Birthday, anniversary, win-back, renewal |
| GET          | `/campaigns/{campaign}/results`               | Sent, delivered, bookings, revenue       |
| GET          | `/message-logs` and `/message-credits`        | Status, cost, balance                    |
| GET/POST     | `/reviews`                                    | Collected feedback                       |

### Website Management

| Method       | Path                                                                                    | Notes                                                           |
| ------------ | --------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| GET/POST/PUT | `/website/pages`, `/posts`, `/menus`, `/sections`, `/banners`, `/redirects`, `/gallery` | Bilingual; gallery items need consent when they show a customer |
| POST         | `/website/publish`                                                                      | Purges caches, updates sitemap                                  |
| GET/PUT      | `/website/settings`                                                                     | Branding, accent contrast check, tag manager IDs                |
| POST         | `/website/domains` and `/domains/{domain}/verify`                                       | DNS verification and TLS status                                 |
| POST         | `/media`                                                                                | Upload into the processing queue                                |

### Tasks, Approvals, Dashboard & Reports

| Method       | Path                                                                  | Notes                                                                                                                                                                      |
| ------------ | --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET/POST/PUT | `/tasks`                                                              | Related entity, due date                                                                                                                                                   |
| GET/POST/PUT | `/approval-workflows`                                                 | Configuration                                                                                                                                                              |
| GET          | `/approvals/inbox`; POST `/approvals/{request}/approve` and `/reject` | Comment required on reject                                                                                                                                                 |
| GET          | `/dashboard?from=&to=&branch=`                                        | KPIs from `prd.md` section 5, alerts                                                                                                                                       |
| GET          | `/reports/{report}`                                                   | `revenue`, `payments`, `appointments`, `no-shows`, `utilization`, `customers`, `retention`, `top-services`, `inventory`, `commission`; PDF and Excel; long reports as jobs |
| GET          | `/exports/{export}`                                                   | Download a finished export                                                                                                                                                 |

---

## 5. Customer Account Endpoints

Tokens with ability `customer`; everything is scoped to the signed-in customer.

| Method | Path                                                         | Notes                     |
| ------ | ------------------------------------------------------------ | ------------------------- |
| GET    | `/me/appointments`                                           | Upcoming and past         |
| POST   | `/me/appointments/{appointment}/reschedule` and `/cancel`    | Within the allowed window |
| GET    | `/me/packages`, `/me/points`, `/me/gift-cards`, `/me/wallet` | Balances and ledger       |
| GET    | `/me/invoices` and `/me/invoices/{invoice}`                  | PDF through a signed URL  |
| GET    | `/me/referral`                                               | Code and rewards          |
| POST   | `/me/reviews`                                                | Feedback after a visit    |

---

## 6. Platform API (`/api/v1/platform`, super-admin only)

| Method       | Path                                      | Notes                                            |
| ------------ | ----------------------------------------- | ------------------------------------------------ |
| GET/POST/PUT | `/tenants`                                | Create and manage businesses                     |
| PUT          | `/tenants/{tenant}/plan`                  | Plan, limits, module toggles, trial dates        |
| POST         | `/tenants/{tenant}/suspend` / `/resume`   | Suspension                                       |
| GET/POST     | `/tenants/{tenant}/subscription-payments` | Subscription ledger                              |
| POST         | `/tenants/{tenant}/message-credits`       | Add credits (adjustments need a reason)          |
| POST         | `/tenants/{tenant}/impersonate`           | Short-lived, audited, with a visible banner      |
| POST         | `/tenants/{tenant}/export`                | Full data export job                             |
| POST         | `/tenants/{tenant}/imports`               | Import monitor: upload, `dry_run`, commit        |
| GET          | `/tenants/{tenant}/domains`               | Domain and TLS status                            |
| GET          | `/tenants/{tenant}/usage`                 | Branches, staff, appointments, storage, messages |

---

## 7. Messaging Providers

- `MessageChannel` implementations: `SmsChannel` (BdBulkSMS), `EmailChannel` (Laravel Mail), `WhatsAppChannel` (WhatsApp Cloud API, pre-approved templates only). All sends are queued with retries. Credits are reserved and deducted in the same transaction that writes the message log; a failed send refunds the reservation. Message logs store template, channel, status and cost, not extra personal data. Sender ID and WhatsApp template approval depend on the provider and local rules; confirm with them.
- **Mobile banking payments** (bKash, Nagad) are recorded with the TrxID and flagged `unverified` until matched with the merchant statement. Direct payment gateway integration is out of scope for v1.
- Webhooks: provider delivery status and WhatsApp inbound status are public web routes, signature-verified, idempotent and processed in a queue.

---

## 8. Dashboards and Real-Time

There is no real-time server in v1. Dashboards refresh on load and every few minutes; the calendar refreshes by light polling (about every 30 seconds) and after the user's own actions; approvals reach users through the in-app inbox, optional SMS, and later push notifications.
