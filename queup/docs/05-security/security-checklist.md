# Queup — Security Checklist

**Workstream:** 05-security
**Status:** Complete (v1 — living document; gate every release)
**Depends on:** `auth-design.md`, `gdpr-compliance.md`, `pci-dss-notes.md`, `04-technical/*`

This is the pre-launch and per-release gate. Every box must be ticked (or explicitly risk-accepted and dated) before shipping to production. Grouped by domain; `[ ]` = to verify.

---

## 1. Authentication & authorization

- [ ] Supabase Auth configured; email verification required before first order / go-live.
- [ ] Password policy enforced client **and** server (≥8 chars, ≥1 number).
- [ ] Google + Apple OAuth providers configured; redirect URIs allow-listed.
- [ ] Access tokens: short TTL (~1h); refresh-token rotation + reuse detection ON.
- [ ] Tokens stored in Expo SecureStore (mobile); never in web `localStorage`.
- [ ] `role` claim is server-set app metadata; not client-editable. Verified by test.
- [ ] Deny-by-default: every route requires JWT unless explicitly marked public.
- [ ] Ownership checks on all vendor/order mutations (no IDOR). Covered by tests.
- [ ] **Admin (manager) accounts require MFA (TOTP);** no admin self-sign-up.
- [ ] Sign-out revokes refresh token and clears SecureStore + device token.

---

## 2. Row-Level Security & data access

- [ ] RLS enabled on `customers`, `vendors`, `orders`, `order_items`, `devices`, `collection_slots`.
- [ ] Customer sees only own profile + own orders (policy tested).
- [ ] Vendor sees only own vendor row, menu, and orders (policy tested).
- [ ] Realtime subscriptions respect RLS (vendor queue / customer tracker leak-tested).
- [ ] Public read policy limited to `active` vendors only.
- [ ] Service-role key used **only** by backend, never shipped to a client.

---

## 3. Payments & PCI (SAQ A)

- [ ] No card fields anywhere in Queup UI — Stripe Payment Sheet only.
- [ ] No PAN/CVV in DB, logs, analytics, or Sentry (payment payloads scrubbed).
- [ ] Stripe **secret** + **webhook** keys are backend env only; publishable key only on clients.
- [ ] `stripe.webhooks.constructEvent` verifies **every** webhook — no skip path (code-reviewed).
- [ ] Webhook handling idempotent by `event.id` (`payment_events`), replay-safe.
- [ ] Order totals recomputed server-side (pence); client totals never trusted (tested).
- [ ] PaymentIntent uses `application_fee_amount` (5%) + `transfer_data.destination`.
- [ ] Refund path on `pending → rejected` verified in Stripe test mode.
- [ ] Annual SAQ A attestation scheduled; Stripe SDK on a current version.

---

## 4. Input validation & injection

- [ ] Zod schema on every route body/query/param; invalid → `400`, no stack trace leak.
- [ ] All DB access via Supabase parameterised queries — no string-concatenated SQL.
- [ ] Enum/domain values (`vendor_mode`, `order_status`, etc.) constrained in DB **and** validated in code.
- [ ] File/image uploads (vendor hero/menu images) validated for type + size; served from object storage, not executed.
- [ ] Order status transitions validated in service **and** by DB trigger (illegal → `409`).

---

## 5. Location rules (Queup-specific, security-relevant)

- [ ] `home_food` vendors never reach any what3words code path (tested — attempted GPS update → `409`).
- [ ] what3words called at most **once** per Segment A location update (verified).
- [ ] Customer GPS never persisted — no customer location table exists (schema-reviewed).
- [ ] Segment A vendor `geom`/`w3w_address` overwritten each update; no history table exists.
- [ ] Background location permission declared for **vendor app, Segment A only** (`app.json` reviewed).
- [ ] Location permission purpose strings present and accurate (iOS + Android).

---

## 6. Transport & network

- [ ] TLS enforced on all API endpoints (HTTP → HTTPS redirect / HSTS).
- [ ] **Certificate pinning** on production client → API calls.
- [ ] CORS locked to known origins for the manager web app.
- [ ] Security headers on web (CSP, X-Content-Type-Options, Referrer-Policy, HSTS).
- [ ] `/healthz` exposes no secrets or internal detail.

---

## 7. Rate limiting & abuse

- [ ] Rate-limit middleware active on all public routes.
- [ ] `/auth/*` ≤ 10/min/IP; `/location/update` ≤ 3/min/vendor; orders/payments ≤ 30/min/user; default 100/min/IP.
- [ ] `429` returns `Retry-After`.
- [ ] what3words usage monitored; alert at 80% of the 25k/month free tier.
- [ ] Collection-code brute force mitigated: `verify-code` rate-limited per vendor/order.

---

## 8. Secrets & configuration

- [ ] `.env` git-ignored; only `.env.example` (placeholders) committed. History scanned for leaked secrets.
- [ ] All required env vars validated at boot (`config/env.ts`); process exits if missing.
- [ ] Secrets injected per environment (dev/staging/prod); no shared prod keys in dev.
- [ ] Key rotation runbook exists; any suspected exposure → immediate rotation.
- [ ] Pre-commit / CI secret-scanning enabled.

---

## 9. Mobile app hardening

- [ ] No secret keys in any client bundle (grep the built bundle to confirm).
- [ ] Expo OTA updates are **signed**; update integrity verified.
- [ ] Jailbreak/root detection considered (recommended for a payment app) — decision recorded.
- [ ] Deep links / URL schemes validated; no open-redirect / arbitrary navigation.
- [ ] Sensitive screens not exposed in app-switcher snapshots where feasible.

---

## 10. Observability & incident response

- [ ] Sentry on client + backend; PII scrubbed from events.
- [ ] Structured logs; no secrets/PANs/tokens in logs (log review).
- [ ] Audit trail for admin actions (approvals, suspensions, refunds).
- [ ] Breach response runbook: detect → contain → assess → **ICO within 72h** if reportable → notify users if high risk.
- [ ] Managed backups verified (Supabase); restore tested.

---

## 11. GDPR / privacy gates (see `gdpr-compliance.md`)

- [ ] Privacy policy presented + explicitly accepted at sign-up (not pre-ticked).
- [ ] Marketing opt-in unbundled, default off.
- [ ] Age gate enforced at sign-up (18+, or 13+ w/ consent).
- [ ] Self-service data export + account deletion implemented; erasure cascade tested (orders anonymised, not lost).
- [ ] DPAs signed with Supabase, Stripe, Firebase, what3words, Sentry.
- [ ] ICO registration / data-protection fee paid.
- [ ] DPIA completed for location processing.
- [ ] Sub-processor list current in the privacy policy; international-transfer safeguards documented.

---

## 12. Regulatory (UK)

- [ ] ICO registered (data-protection fee).
- [ ] FCA: confirmed with solicitor that Queup holding **no** customer funds (Stripe does) means authorisation not required — decision recorded.
- [ ] FSA: platform ToS makes clear food-safety/registration is the vendor's responsibility, not Queup's.

---

## 13. Release gate

A release ships only when: every box above is ticked or has a **dated, named risk acceptance**; `tsc --noEmit`, lint, and tests are green; and payment + location paths have been exercised on a real device / Stripe test mode.

**Sign-off:** ______________________  **Date:** ____________
