# ADR-0001: v1 stack, scope, and explicit non-goals

- **Status**: accepted
- **Date**: 2026-10-14
- **Deciders**: maintainer + agent-assisted grilling session

This ADR captures the v1 decisions for `TheZenMind07/hotel-booking`. Future ADRs will record narrower decisions (deployment plumbing, schema migrations, Stripe integration, etc.) on top of this base.

## Context

The repo started empty. We needed a single-property hotel booking app, end-to-end, scoped to a believable MVP, with decisions that future engineers (and future agents) can read on the first session instead of re-deriving from history.

## Decisions

### Product

- **One traveller-facing flow**: `/` → `/search` → `/book/[roomId]` → `/confirmation/[ref]`.
- **Single property**, single operator. Multi-property / multi-tenant is v2.
- **Admin is read-only in v1**: `/admin/bookings` (list) and `/admin/bookings/[ref]` (detail). No room CRUD, no cancel button. If a cancellation or room edit is required pre-launch, the operator edits the Postgres row directly.

### Stack

- **Framework**: Next.js (App Router), TypeScript.
- **DB**: Postgres.
- **ORM**: Prisma.
- **Styling**: Tailwind CSS + shadcn/ui.
- **Deployment**: Vercel + Neon (free tier). Migration to AWS later is mostly mechanical — Postgres stays Postgres.
- **Auth in v1**: none for travellers (email collected at booking time). Admin gated by an `ADMIN_EMAILS` env-var allowlist + a signed cookie claim; in dev, a `?admin=1` query string sets the claim when the requester email matches.

### Data model

- **Per-stay bookings** with explicit range overlap checks at submit time.
- **Single unit per room type** in v1: each bookable room is its own `Room` row. Multi-unit-per-type is a v2 schema change.
- **`Room` fields**: id, name, description, image URL, nightly_rate_cents, max_guests, bed_count.
- **`Booking` fields**: id, booking_ref (8-char nanoid, unique), room_id, guest_email, guest_name, guests (int, record-keeping only), check_in_date, check_out_date, total_cents, idempotency_key (unique), created_at.
- **No amenities column on `Room` in v1**.
- **No cancellation state on `Booking` in v1**.
- **Currency is USD only**, stored as integer cents; the formatter has an implicit `USD` constant.

### Booking flow

1. `/search?check_in=...&check_out=...` filters available rooms.
2. `/book/[roomId]` shows the room, a date picker, and a guest-details form.
3. On submit, the server action:
   - Re-checks availability inside a `prisma.$transaction`.
   - Inserts a `Booking` row with a server-stored `idempotency_key` (UUID generated client-side, unique index).
   - Returns the booking reference.
4. `/confirmation/[ref]` shows the booking reference and details.

### Validation

- **Strictly-after check-out**: `check_out_date > check_in_date`. No same-day bookings.
- **No past dates**: today onward.
- **Tomorrow-asm-earliest check-in**: earliest possible check-in is `today + 1`.
- **Max 30 nights per booking**.
- All enforced by a single Zod schema shared by the client form and the server action.

### Pricing

- **Flat nightly rate per `Room`**. Total = `nights × nightly_rate_cents`. Stored on the `Booking` row at the moment of booking so later rate changes don't retroactively mutate confirmed bookings.
- `guests` on the booking is record-keeping only and is validated against `room.max_guests`. No extra-guest surcharge in v1.

### Booking reference

- **8-character nanoid** on a unique `Booking.booking_ref` column. The internal `id` stays as a UUID so the reference can be regenerated if needed.

### Concurrency

- v1 **relies on transactional re-checks** at submit, not on a database-level exclusion constraint. The overlap predicate is `existing.check_in < new.check_out AND existing.check_out > new.check_in`.
- v2 path: introduce `daterange` columns + a GiST exclusion constraint under `btree_gist`. The query path stays the same; the schema picks up range types and an `EXCLUDE USING gist` clause. ADR to be written when this lands.

### Idempotency / double-submit

- The booking form has a client-generated UUID idempotency key, sent with the submission and stored on the `Booking` row with a unique index. A retry of the same form returns the existing booking; the server action does not create a second row.
- The submit button is disabled while the server action is pending (`useActionState().isPending`). Together, these handle both the user-double-click case and the flaky-network retry case.

### Observability

- **Structured JSON logs to stdout** (`console.log(JSON.stringify({...}))`). Vercel tails stdout with basic filtering for free. No third-party platforms in v1.

### Testing

- **Unit tests (Vitest)** for the overlap predicate and the date-validation Zod schema.
- **One Playwright happy-path test** covering `/search` → `/book` → `/confirmation`.
- No snapshot tests in v1.

## Out of scope for v1

These are explicit non-goals, agreed during the design session, recorded here so they don't drift back in without an ADR:

- **Payments**: stubbed (no `Payment` table in v1; the booking jumps straight to `confirmed`). Stripe (PaymentIntents with `metadata.booking_id`, webhook for status updates) is planned for v2.
- **Email receipts**: on-screen confirmation only.
- **User-facing cancellation flow**: not built.
- **Multi-currency / FX**.
- **Multi-property / multi-tenant**.
- **Reviews, ratings, recommendations**.
- **i18n / localization**.
- **Native mobile app**. Responsive web only.
- **Analytics / tracking**.
- **SSO, SAML, social login**. Only the `ADMIN_EMAILS` allowlist.

## Consequences

Positive:

- The v1 surface area is small enough to ship and iterate quickly. Most of the schema and route shape carries over to v2 with additive changes.
- Locking the booking data model now gives the Stripe-in-v2 work a defined handoff surface (the `PaymentProvider` interface and `Booking.status` field).
- Recording this ADR means future agents re-derive less from history.

Negative / risks:

- The `Booking` table lacks a `status` column in v1. v2 will need a migration to add `status` and a `Payment` table, both safe additive changes if we keep `idempotency_key` immutable.
- Without database-level exclusion constraints, two near-simultaneous submit requests can race past the application-level check. Mitigation: the application-level check runs inside a `prisma.$transaction` and the `idempotency_key` unique index catches accidental duplicates. v2 trades this for a GiST exclusion constraint.
- Admin access is gated by a signed cookie, not a real session. Acceptable for a single-operator v1; not acceptable if admin duties are delegated.

## Follow-ups to track

- ADR for the **Stripe / PaymentIntents** integration (v2).
- ADR for the **Postgres exclusion constraint** upgrade when concurrency demands it.
- ADR for **room CRUD in admin** when the operator wants to stop editing rows directly.
- ADR for **email receipts** (Resend vs SES vs console-log-stub) when we pick a provider.
