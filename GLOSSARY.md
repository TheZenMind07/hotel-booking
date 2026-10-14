# Glossary

Terms as used in this repo. Names of these should be the source of truth in issues, PRs, and chat — drift to synonyms is a signal that the concept needs better naming or a glossary update.

## Domain

- **Booking** — a confirmed reservation for one guest's stay at one `Room` over a date range. Sub-concepts below.
- **BookingReference** — the 8-character nanoid on `Booking.booking_ref`. The customer-facing identifier. Distinct from `Booking.id` (UUID), which is the internal handle.
- **Room** — a single, bookable physical unit. In v1 there is exactly one unit per "room type"; the `Room` row conflates type and unit. Future: split into `RoomType` (the abstract type) and `Room` (the physical instance) when we go multi-unit-per-type.
- **RoomType** — in v1, this concept lives *inside* a single `Room` row (name, bed_count, max_guests). In v2 it becomes its own table when multi-unit inventory is introduced.
- **SearchWindow** — a `(check_in_date, check_out_date)` tuple used to query availability. Validation rules are pinned in `docs/adr/0001-v1-stack-and-scope.md`.
- **OverlapPredicate** — the rule `existing.check_in_date < new.check_out_date AND existing.check_out_date > new.check_in_date` used to detect conflicting bookings. Holds for both application-level checks (v1) and `daterange` exclusion constraints (v2).
- **Guest** — the person named on the `Booking` (`guest_name`, `guest_email`). Not a user account in v1.
- **AmountCents** — integer cents in USD. All monetary values in the schema use this, never floats. The formatter has an implicit `USD` constant in v1.
- **NightlyRate** — the integer cents per night for a `Room`. Total = `nights × NightlyRate` at the moment of booking.
- **PaymentStub** — the v1 placeholder for "this booking has no payment yet." Replaced by a real `Payment` row tied to a Stripe `PaymentIntent` in v2.

## Operational

- **Admin** — an operator email listed in the `ADMIN_EMAILS` env var. In v1 they have read-only access to `/admin/bookings`. Gate is a signed cookie (no NextAuth).
- **Operator** — synonym for Admin in casual usage. We prefer "Admin" in code and ADRs.
- **IdempotencyKey** — a client-generated UUID stored on `Booking.idempotency_key` with a unique index. Two identical submissions return the same booking rather than creating a second row.

## Anti-terms (do not introduce)

These are names we have explicitly chosen **not** to use, so that future PRs don't drift into synonymy:

- `Order` — we say **Booking**. "Order" implies payment/line-items; a `Booking` can exist without a payment.
- `Listing` or `Property` — we have a single property. Multi-property is v2; when we get there, prefer **Property** as the new name and migrate.
- `Customer` — prefer **Guest**. We use `Customer` only if Stripe webhook payloads require it.
- `PricingPlan` / `RatePlan` — v1 has only a flat `NightlyRate`. Don't add a `PricingPlan` table without an ADR.
- `Inventory` / `Stock` — v1 has one unit per room. Don't introduce an inventory-count concept without an ADR.
