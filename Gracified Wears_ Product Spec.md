# Gracified Wears: Product Spec (v1 draft)

Scriptural customization of wears, ordered by customers, produced by vendors, run by one admin.

---

## 1. Roles

| Role | How they get in | What they do |
| --- | --- | --- |
| **Customer** | Signs up with email OTP, then onboarding (name, phone) | Browses, customizes, pays, tracks orders, raises disputes |
| **Vendor** | Signs up (email, phone, name), then onboarding (certificate, company name, location) | Accepts or declines assigned orders, updates production status, chats with admin, tracks payouts |
| **Admin** | Seeded account only, no signup | Runs everything: orders, vendors, statuses, payouts, disputes |
| **Logistics** | Later | Can scan barcodes for delivery. Not in v1 |

**Sign-in:** email OTP at launch. Phone number is collected in onboarding. WhatsApp OTP can be added later if the running cost is acceptable.

---

## 2. Phases

1. **Phase 1: Core ordering.** Catalog, customization, cart, checkout, payment, order success email, admin order list and status updates, customer My Orders.
2. **Phase 2: Vendors.** Vendor signup, onboarding, admin approval, assignment, accept or decline, vendor status updates, delivery estimate, barcode scan, status emails.
3. **Phase 3: Money and disputes.** Payout rules and approvals, vendor payments view, disputes, refunds, chat and negotiation.

---

## 3. Catalog and customization

**Wear fields:** category (Hoodie, Jogger, Sweatshirt, Shirt, and more), price, images, maker (hidden until decided), default inscription, and variants (color, size, material). Each variant can carry its own price adjustment.

**Inscription modes:**

| Mode | Meaning | Available to |
| --- | --- | --- |
| Default | Admin-curated inscription for that product, shown on its preview image | Personal, Society |
| Custom | Customer writes verse text and reference (80 character limit, placement and print color chosen) | Personal, Society |
| Random | Each piece draws from the admin-uploaded pool | **Society only** |

**Random draw rule:** the draw happens at order time. The result is saved on each order item, so the customer, admin and vendor all see the same fixed list. It is never redrawn.

**Admin inscription pool:** upload via CSV or paste (text and reference per row), edit or remove entries, and keep multiple pools if wanted.

---

## 4. Order types

|  | Personal | Society |
| --- | --- | --- |
| Buyer | One person | A group (church, class, association) |
| Inscription | Default or Custom | Default, Custom, or Random |
| Same or different per piece | n/a | Same Inscription or Random |
| Sizes | One size | Size breakdown (e.g. 5 S, 10 M, 8 L) |
| Quantity | Entered by customer | Sum of the size breakdown |

**Fulfilment:** Delivery (address required) or Pickup. Pickup implies a physical store and is shown as "coming soon" until that exists.

**Checkout collects:** quantity, order type, inscription choice, sizes, delivery or pickup, address, and customer details.

---

## 5. Order lifecycle

Two status layers exist so the vendor cannot change what the customer sees.

**Vendor-side status** (set by the vendor): `ASSIGNED`, `ACCEPTED`, `DECLINED`, `IN_PROGRESS`, `READY`

**Customer-facing status** (set by the admin, and the only one the customer sees):

| Status | Set by | Meaning |
| --- | --- | --- |
| Pending payment | System | Order created, payment not confirmed |
| Paid | System (payment webhook) | Payment confirmed, awaiting assignment |
| Assigned | Admin | Sent to a vendor |
| In progress | Admin | Vendor is producing it |
| Ready | Admin | Produced, awaiting delivery |
| Delivering | Admin | On its way |
| Completed | System or admin | Barcode scan confirms delivery |
| Disputed | System (on dispute) | Dispute open |
| Cancelled | Admin | Order cancelled |

**How the layers connect:**

- The vendor marks IN_PROGRESS or READY, and the admin sees this on the order card.
- The admin then moves the customer-facing status, which triggers the customer email.
- If a vendor declines, the order returns to the admin as unassigned to be re-assigned, and the admin is notified by email.

**Delivery estimate:** calculated from quantity and delivery location using an admin-editable table (production days by quantity tier plus transit days by location zone). It is generated and communicated when the status is set to In progress. The order success email says "delivery estimate to follow".

---

## 6. Money flow

**Customer to platform:** the customer pays in full at checkout through the platform (Paystack or Flutterwave suggested).

**Platform to vendor:**

| Stage | Trigger | Amount | Approval |
| --- | --- | --- | --- |
| Upfront payout | Order set to In progress | 50% of the vendor payout by default, or an admin-set value for that order | Admin approval, with authorization |
| Balance payout | 2 days after delivery, if no dispute is open | Remainder | Admin approval, with authorization |

**Vendor payout amount:** set by the admin per order at assignment and kept separate from the customer's price (the platform keeps the difference).

**Vendor payment status:** `PENDING`, `PAID`, `DUE`.

**Authorization flow:** the admin approves a payout by confirming their password and then entering an emailed OTP. Every approval is written to an audit log (who, when, amount, order).

---

## 7. Delivery and barcode

- Each order gets an order code and a barcode, sent in the customer's success email.
- Delivery can be done by the platform or by the vendor.
- **Who can scan:** the admin, the vendor assigned to that order, and logistics later.
- A successful scan confirms delivery, sets Completed, and starts the dispute window and balance timer.
- Pickup uses the same barcode for confirmation at the store later.

---

## 8. Disputes

**Customer dispute**

- Can be raised within 24 hours of delivery confirmation.
- The balance payout (2 days after delivery) is held while the dispute is open.

**Vendor dispute**

- Can only be about payment (for example, a missing or incorrect payout).

**Admin outcomes**

| Outcome | Rule |
| --- | --- |
| Full refund | Only available when the vendor's balance has not been paid out. The admin sets the upfront percentage on the order card at assignment, and if 100% was paid upfront, full refund is not possible |
| Partial refund | Admin enters the amount, capped at what the customer paid. The vendor's balance is reduced to match |
| Reject | Dispute closed, payouts proceed |

Each dispute has a thread, status (`OPEN`, `UNDER_REVIEW`, `RESOLVED`), outcome, and notes. The customer and admin get emails on open and resolution.

---

## 9. Vendor onboarding and approval

1. Vendor signs up (email, phone, name) and verifies email OTP.
2. Onboarding form: company name, location, certificate upload, and other details as needed.
3. Status becomes `PENDING_APPROVAL`. The vendor sees an empty dashboard with **no navigation** available.
4. The admin reviews and chooses Approve or Reject (with reason).
5. The vendor receives an email either way. Approval unlocks navigation.

---

## 10. Screens by role

**Customer**

- Signup, onboarding, profile with saved addresses
- Catalog, product page with live preview, cart, checkout
- My Orders with detail, status timeline, barcode, and dispute button (while the 24-hour window is open)

**Vendor**

- Signup, onboarding, pending-approval empty state
- Assigned orders list and order card (accept or decline, status buttons, per-order chat with the admin for negotiation)
- General chat with the admin
- Payments (`PENDING` / `PAID` / `DUE`), payment dispute, barcode scanner

**Admin**

- Orders list (filters by status, type, vendor) and order details
- Status update, delivery estimate and tracking, vendor assignment with payout settings
- Vendor management (approval queue, profiles) and vendor communication (per-order and general)
- Payments (customer to platform, platform to vendor, approval queue with authorization)
- Dispute management, barcode scan, inscription pool and default inscription management, estimate table settings

---

## 11. Notifications

| Event | Customer | Admin | Vendor |
| --- | --- | --- | --- |
| Order placed | Success email (order code, barcode, details, "estimate to follow") | Full order notice |  |
| Payment received | Payment email | Payment email |  |
| Order assigned |  |  | Assignment email |
| Vendor declines |  | Decline notice |  |
| In progress | Status email with delivery estimate |  |  |
| Ready | Status email |  |  |
| Delivering | Status email |  |  |
| Completed | Status email |  |  |
| Vendor approved or rejected |  |  | Decision email |
| Payout approved or paid |  |  | Payout email |
| Dispute opened or resolved | Yes | Yes | Yes (payment disputes) |
| Chat message |  | In-app and email | In-app and email |

---

## 12. Data model

`User` (role, email, phone) · `CustomerProfile` · `Address` · `VendorProfile` (company, location, certificate, approval status) · `Product` · `Variant` (color, size, material, price adjustment) · `DefaultInscription` · `InscriptionPool` and `PoolEntry` · `Cart` and `CartItem` · `Order` (type, fulfilment, customer-facing status, code, barcode, estimate) · `OrderItem` (variant, size, resolved inscription, placement, print color) · `Assignment` (vendor, vendor status, payout total, upfront %) · `StatusHistory` · `Payment` · `Payout` (stage, amount, status, approver) · `Refund` · `Dispute` · `Message` and `Thread` · `ScanEvent` · `EmailLog` · `AuditLog` · `OTP`

---

## 13. Suggested stack

A TypeScript web app (Next.js) with PostgreSQL, Paystack for payments, an email provider (Resend or similar), barcode generation, and file storage for certificates and images. Best built in a real codebase in phases, for example with Claude Code.

---

## 14. Assumptions to confirm

1. The vendor payout is set separately from the customer's price, and the platform keeps the difference.
2. "Within 24 hours after delivery" means the dispute window closes 24 hours after the scan.
3. Full refund eligibility depends on the upfront percentage set at assignment (100% upfront means no full refund).
4. Delivery estimate rules (production days by quantity, transit days by zone) will be supplied by you, and placeholder values will be used until then.
5. One currency (₦) at launch.