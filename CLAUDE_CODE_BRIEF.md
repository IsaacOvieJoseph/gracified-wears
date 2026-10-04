# Gracified Wears: Build Brief for Claude Code

Save this as `CLAUDE.md` (or `BRIEF.md`) in the repo root. Work through the build order one step at a time, and stop after each step to summarize what changed and what to test. Ask before deviating from anything here.

## 1. What we're building

A marketplace for scriptural custom wears (hoodies, joggers, sweatshirts, shirts). Customers order and pay the platform, a single admin assigns orders to vendors, vendors produce them, and the admin controls the status the customer sees.

Roles: **Customer**, **Vendor**, **Admin** (one seeded account, no signup). Logistics comes later.

This brief covers **Phase 1** in full. Phases 2 and 3 are summarized in section 10 so the schema is ready for them.

## 2. Stack and layout

| Layer | Choice |
|---|---|
| Frontend | Next.js (TypeScript) on Vercel |
| Backend | Node.js, Express, TypeScript on Render |
| Database | MongoDB Atlas via Mongoose |
| Payments | Paystack |
| Email | Brevo, behind an `EmailProvider` interface so Resend can replace it later |

```
/apps/web        Next.js app (customer + admin UI)
/apps/api        Express API
/packages/shared types, enums, constants (statuses, roles) used by both
/scripts         seed-admin, import-pool
```

Currency is ₦. Store money as integers in **kobo**.

## 3. Rules that must hold everywhere

1. **Prices are calculated on the server only.** Never trust a price from the client.
2. **Customer-facing status is set only by the admin** (or by the system for payment and delivery events). Vendors never change it.
3. **Random inscriptions are drawn once, at order creation, and saved on each piece.** Never redraw.
4. **Paystack webhooks are verified** by HMAC SHA512 signature (`x-paystack-signature`) and are idempotent by payment reference.
5. Every admin money action (Phase 3) needs password plus emailed OTP and writes to the audit log.
6. Pickup exists in the UI as "coming soon" and cannot be selected.
7. Validate all input (zod), rate-limit OTP and login endpoints, and keep secrets in env vars only.

## 4. Order model (the core)

**Order types**
- **Personal:** one piece, one size, inscription Default or Custom.
- **Society:** a size breakdown (e.g. `{S:5, M:10, L:8}`), inscription Default, Custom (same on every piece), or Random (each piece draws from the admin pool).

**Inscription choices:** Default (admin-curated per product), Custom (text up to 80 characters, reference up to 30), Random (society only).

**Stored on the order:** `pieces: [{ size, inscription: { text, reference, source: "default"|"custom"|"random" } }]`, plus placement and print color.

**Random draw:** shuffle the pool and take pieces in sequence. If the pool is smaller than the piece count, reshuffle and continue, so repeats happen only when unavoidable.

**Customer-facing statuses:** `PENDING_PAYMENT`, `PAID`, `ASSIGNED`, `IN_PROGRESS`, `READY`, `DELIVERING`, `COMPLETED`, `DISPUTED`, `CANCELLED`

**Phase 1 transitions (admin):** `PAID → IN_PROGRESS → READY → DELIVERING → COMPLETED`, and `CANCELLED` from any status before COMPLETED. `ASSIGNED` and `DISPUTED` are in the enum but unused until Phases 2 and 3. Record every change in `StatusHistory` (from, to, by, at).

**Order code:** `GW-YYMMDD-XXXX` (random base32 suffix, unique).

**Delivery estimate:** computed when the admin sets `IN_PROGRESS`, from quantity and delivery location, using an admin-editable `EstimateSettings` document (production days by quantity tier, transit days by location zone). Ship with placeholder values. The order success email says "delivery estimate to follow", and the `IN_PROGRESS` email carries the real estimate.

## 5. Collections

Build these Phase 1 collections fully:
`User` (role, email, phone, name, onboarded) · `Otp` (hashed, expiry, attempts) · `Address` · `Product` (name, category, images, basePrice, maker (hidden), defaultInscription) · `Variant` (product, color, size, material, priceAdjustment) · `InscriptionPool` and `PoolEntry` (text, reference) · `Cart` · `Order` · `StatusHistory` · `Payment` (reference, amount, status, raw webhook) · `EstimateSettings` · `EmailLog`

Create these as schemas only, with no routes yet, so Phases 2 and 3 need no migrations:
`VendorProfile` · `Assignment` · `Payout` · `Refund` · `Dispute` · `Thread` and `Message` · `ScanEvent` · `AuditLog`

## 6. Phase 1 features

**Customer**
- Email OTP login, then onboarding (name, phone), then saved addresses
- Catalog filtered by category, product page with live garment preview (port the SVG previewer from `gracified-wears.html`, the reference prototype)
- Personal and Society ordering flows as described in section 4
- Cart, checkout (delivery only), Paystack payment, My Orders with status timeline and barcode

**Admin** (seeded login)
- Orders list (filter by status, type, date) and order detail (pieces with inscriptions, customer, address, payment, history)
- Status update control that follows the allowed transitions
- Product, variant and default-inscription management with image upload
- Inscription pool upload (CSV with `text,reference` columns) and edit
- Estimate settings editor

**Barcode:** Code 128 image of the order code, generated server-side and shown in My Orders and the success email.

## 7. Emails (Phase 1)

All through the `EmailProvider` wrapper, logged to `EmailLog`, with a retry on failure.

| Event | To | Content |
|---|---|---|
| Login OTP | Customer | 6-digit code, 10 minute expiry |
| Order paid | Customer | Order code, barcode, full order details, "delivery estimate to follow" |
| Order paid | Admin | Everything needed to act: customer, contact, address, pieces and inscriptions, payment |
| Payment received | Customer and Admin | Amount, reference |
| `IN_PROGRESS` | Customer | Status update plus delivery estimate |
| `READY`, `DELIVERING`, `COMPLETED`, `CANCELLED` | Customer | Status update |

## 8. API outline

```
POST /auth/request-otp        POST /auth/verify-otp       POST /auth/logout
GET/PATCH /me                 CRUD /me/addresses
GET /products  GET /products/:id
POST /cart  ...               POST /orders  (validates, prices, draws random, creates PENDING_PAYMENT)
POST /orders/:id/pay          (initializes Paystack transaction, returns authorization URL)
POST /webhooks/paystack       (verify signature, mark PAID, send emails)
GET /me/orders  GET /me/orders/:id
-- admin --
GET /admin/orders  GET /admin/orders/:id  PATCH /admin/orders/:id/status
CRUD /admin/products  /admin/variants
POST /admin/pool/import  CRUD /admin/pool
GET/PUT /admin/estimate-settings
```

## 9. Build order and "done" criteria

1. **Setup:** monorepo, TypeScript, lint, env validation, Mongo connection, `seed-admin` script (credentials from env). *Done when the admin exists and the API boots.*
2. **Auth:** email OTP, httpOnly JWT cookie, role guards, onboarding. *Done when a new customer can sign in and complete onboarding.*
3. **Admin catalog:** products, variants, default inscription, pool CSV import. *Done when the admin can create a complete product.*
4. **Customer catalog and preview:** list, filter, product page, live preview. *Done when preview matches chosen color, text and placement.*
5. **Cart and order creation:** personal and society flows, server pricing, random draw. *Done when the tests below pass.*
6. **Paystack:** initialize, redirect, webhook, payment verification on return. *Done when a test-mode payment marks the order PAID exactly once.*
7. **Emails and barcode:** provider wrapper, templates, barcode. *Done when all Phase 1 emails send in a test run.*
8. **Admin orders:** list, detail, status updates with emails and estimate. *Done when status changes follow the transitions and email the customer.*
9. **My Orders:** list, detail, timeline, barcode. *Done when a customer sees live status.*

**Tests required:** pricing (variants and adjustments), random draw (no repeats until the pool is exhausted, correct count), size breakdown totals, webhook signature and idempotency, status transition guard.

## 10. Later phases (do not build yet)

**Phase 2: Vendors.** Signup (email, phone, name), onboarding (company name, location, certificate), admin approval (vendor sees an empty dashboard with no navigation until approved, and gets an email either way). Admin assigns an order with payout settings. Vendor accepts or declines (a decline returns the order to the admin to reassign, with email). Vendor sets `IN_PROGRESS` and `READY` on a vendor-side status, the admin sees it and sets the customer-facing status. Barcode scan by admin or the assigned vendor confirms delivery.

**Phase 3: Money and disputes.** Customer pays the platform in full. The vendor payout is set per order. 50% by default (or an admin value) is released at `IN_PROGRESS`, and the balance 2 days after delivery if no dispute is open. Every payout needs admin approval with password plus OTP, and goes to `AuditLog`. Vendor payment status is `PENDING`, `PAID` or `DUE`. Customers can dispute within 24 hours after delivery, and vendors can dispute only payments. The admin can issue a full refund (only if the vendor balance hasn't been paid out, so not when 100% was paid upfront), a partial refund (admin enters the amount), or reject. Per-order chat and general chat between admin and vendor.

## 11. Environment variables

```
MONGODB_URI  JWT_SECRET  ADMIN_EMAIL  ADMIN_PASSWORD
PAYSTACK_SECRET_KEY  PAYSTACK_PUBLIC_KEY  PAYSTACK_CALLBACK_URL
BREVO_API_KEY  EMAIL_FROM  ADMIN_NOTIFY_EMAIL
APP_URL  API_URL  STORAGE_* (image uploads)
```

## 12. First prompt to give Claude Code

> Read CLAUDE.md fully. Start with step 1 of the build order only: set up the monorepo, TypeScript, env validation, the Mongo connection, and the seed-admin script. Create all Phase 1 schemas and the empty Phase 2/3 schemas listed in section 5. Stop and summarize when step 1 is done.
