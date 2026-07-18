# Queup — Claude Code Instructions

**Workstream:** 04-technical
**Status:** Complete (v1)
**Companion to:** root `CLAUDE.md` (read automatically every session)

This document is the operating manual for using Claude Code to build Queup. The root `CLAUDE.md` holds the always-loaded rules; this file holds the *how-to-work* guidance that would make `CLAUDE.md` too long (keep that file under 100 lines).

---

## 1. Golden rules (never violated by generated code)

These mirror the "YOU MUST" block in `CLAUDE.md`. Claude Code enforces them in every change:

1. **Validate Stripe webhook signatures** with `stripe.webhooks.constructEvent` — never skip.
2. **Never commit `.env`** — secrets are backend-only, git-ignored, documented in `.env.example`.
3. **Never call what3words for `home_food` vendors** — they use postal addresses only.
4. **Never call what3words more than once per Segment A location update.**
5. **Enable PostGIS** in Supabase before running migrations.
6. Declare `expo-location` background permission in the **vendor app** `app.json`, **Segment A only**.
7. Run **`tsc --noEmit`** after every TypeScript change.
8. **Test location features on a real device** — simulators don't replicate GPS.
9. **Never trust client totals** — recompute server-side in pence.
10. **Prices are integer pence**; order status is one-directional.

---

## 2. Recommended sub-chats for this workstream

Split the build into focused Claude Code sessions so each has a tight context:

| Sub-chat | Builds |
|---|---|
| **Backend completion** | `notifications.service.ts`, `auth.middleware.ts`, `rateLimit.middleware.ts`, all migrations, `vendors_within_radius` RPC, `geosearch.service.ts` |
| **Vendor app screens** | Onboarding → Stripe Connect → Menu builder → Go Live (Segment A GPS) → Order queue (realtime) → Payouts |
| **Customer app screens** | Splash → Auth → GPS → Segment selector → Discovery (A map / B list) → Profile → Basket → Checkout → Tracker |
| **Manager dashboard** | Vendor mgmt, order overview, event config, analytics, support tools |

Attach at the start of each: root `CLAUDE.md`, `04-technical/architecture.md`, `04-technical/database-schema.md`, `04-technical/api-reference.md`, and the relevant `02-customer-journey/*` flow.

---

## 3. Workflow per change

1. **Read before writing.** Inspect the existing file structure and the relevant doc before generating new files — match the layering (`routes → services → db`) and naming conventions in `code-standards.md`.
2. **Types first.** If the change touches a domain shape, update `shared/types/index.ts` (and the matching DB `CHECK`) before the consumers.
3. **Generate the code.**
4. **`tsc --noEmit`** in each affected package. Fix every type error before moving on.
5. **Test.** Run unit tests; for a mobile change, `npx expo start` and exercise it — don't assume it works. For location/payments, use a real device / Stripe test mode.
6. **Migrations, never hand-edits.** Any DB change is a new numbered migration in `backend/src/db/migrations/`. Never edit the schema directly and never edit a shipped migration.

---

## 4. Location rule in code (the #1 correctness trap)

Every location-touching change must branch on segment:

```ts
if (vendor.location_type === "w3w") {
  // Segment A: background GPS, w3w conversion, PostGIS radius search
} else {
  // Segment B (postal): fixed address, postcode-centroid distance, NO GPS, NO what3words
}
```

- `location.service` / `location` route: reject a GPS update from a `postal` vendor (`409`).
- `geosearch.service.ts`: PostGIS `ST_DWithin` for `w3w`; postcode-centroid distance for `postal`.
- Vendor app: request background location for Segment A modes only.
- Customer app: two distinct vendor-card components (w3w badge vs street address).

If a diff adds a what3words call reachable from a `home_food` path, it is wrong — stop and fix.

---

## 5. Payments in code

- Server builds the PaymentIntent: `amount = order.total_pence`, `application_fee_amount = order.platform_fee_pence` (5%), `transfer_data.destination = vendor.stripe_account_id`.
- The client uses the Stripe Payment Sheet only — Queup never receives card data (keeps SAQ A scope; see `05-security/pci-dss-notes.md`).
- The webhook handler verifies the signature and is idempotent by `event.id` (`payment_events`).

---

## 6. What is built vs. what remains

**Built:** backend entry, Supabase client, routes (vendors/orders/payments/location), w3w service, shared types + utils.

**Remaining:** `notifications.service.ts`, `auth.middleware.ts`, `rateLimit.middleware.ts`, all SQL migrations, `vendors_within_radius` RPC, `geosearch.service.ts`, every Expo screen (both apps), the manager dashboard.

Keep this list current — when a sub-chat finishes a piece, move it from "remaining" to "built" in both this file and `CLAUDE.md`.

---

## 7. Verifying a Claude Code change before committing

Definition of done (from `code-standards.md` §10): `tsc --noEmit` clean in every affected package · lint clean · unit/integration tests pass · location & payment paths exercised on a real device / Stripe test mode · shared-type changes reflected everywhere + in the DB constraint · docs updated if the API/schema/core-rule changed.

Only then commit, with a conventional-commit message, and push.
