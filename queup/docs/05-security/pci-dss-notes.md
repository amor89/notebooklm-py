# Queup — PCI DSS Notes

**Workstream:** 05-security
**Status:** Complete (v1 — scoping basis; confirm SAQ with your acquirer/Stripe)
**Depends on:** `04-technical/architecture.md`, `04-technical/api-reference.md` (§Payments), `06-integrations/stripe-connect.md`
**Standard:** PCI DSS v4.0

> This defines Queup's PCI scope and the controls that keep it minimal. Confirm the exact SAQ type with Stripe / your acquiring bank before launch.

---

## 1. Scope determination — Queup is SAQ A

**Queup never touches raw cardholder data.** All card entry happens inside Stripe's SDK/UI:
- **Mobile:** Stripe Payment Sheet (iOS/Android) collects and tokenises the card on-device; only a `client_secret` and confirmation flow back through Queup.
- The card number (PAN), expiry, and CVV **never** reach the Queup client bundle, the Queup backend, Queup logs, or the Queup database.

Because Queup fully outsources card handling to a PCI-DSS-validated Level 1 provider (Stripe) and never electronically stores, processes, or transmits cardholder data on its own systems, it qualifies for **SAQ A** — the lightest PCI compliance tier for e-commerce/mobile merchants that outsource all cardholder-data functions.

| What Queup handles | PCI status |
|---|---|
| Card PAN / expiry / CVV | **Never** — Stripe only |
| `client_secret`, `payment_intent_id` | Not cardholder data — safe to hold |
| Stripe customer / Connect account ids | Not cardholder data |
| Order amounts (pence) | Not cardholder data |

---

## 2. Controls that keep Queup in SAQ A

Any of these breaking would risk pulling Queup into a heavier SAQ (A-EP / D). They are hard rules:

1. **No card fields in Queup UI.** Payment input is always the Stripe Payment Sheet — Queup never renders its own PAN/CVV inputs, and never proxies them.
2. **No card data in transit through Queup.** The backend creates PaymentIntents (amount + fee + destination) and receives only tokens/ids — never card numbers.
3. **No card data at rest.** No schema column, log line, analytics event, or Sentry breadcrumb ever contains a PAN/CVV. Telemetry scrubs payment payloads (`payment_events.payload_digest` is a hash, not the raw payload).
4. **Secret keys server-side only.** `STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET` live in backend env vars, never in a client bundle. Clients hold only the **publishable** key.
5. **TLS everywhere.** All client↔backend and backend↔Stripe traffic is HTTPS/TLS.

---

## 3. Webhook signature validation (mandatory)

Every Stripe webhook is verified with `stripe.webhooks.constructEvent(rawBody, sig, STRIPE_WEBHOOK_SECRET)` — **never skipped, no bypass path.** This prevents forged payment-confirmation events (which could otherwise mark orders paid and generate collection codes fraudulently). Handling is also idempotent by `event.id` (`payment_events`) so replays are safe. This is enforced as a "YOU MUST" rule in `CLAUDE.md` and checked in `security-checklist.md`.

---

## 4. Stripe Connect & split payments

- Payments use **Connect** with `application_fee_amount` (5% platform fee) and `transfer_data.destination = vendor Connect account`.
- **Vendor KYC/identity** is handled entirely by Stripe's Connect onboarding — Queup never collects or stores vendor bank details or identity documents.
- Payouts to vendors are Stripe's responsibility; Queup **does not hold customer funds**, which also lightens the FCA posture (see `gdpr-compliance.md` §regulatory / workstream 07).

---

## 5. Responsibilities split

| Requirement area | Stripe (validated Level 1) | Queup (SAQ A merchant) |
|---|---|---|
| Card data capture, storage, transmission | ✓ | — (never) |
| Tokenisation & vault | ✓ | — |
| Network/segmentation of CDE | ✓ | n/a (no CDE) |
| Keeping card data out of its systems | — | ✓ |
| Protecting Stripe secret/webhook keys | — | ✓ |
| TLS on Queup endpoints | — | ✓ |
| Serving current Stripe SDK (no deprecated integrations) | shared | ✓ keep SDK current |
| Annual SAQ A attestation | — | ✓ |

---

## 6. Ongoing obligations

- **Complete an annual SAQ A** self-assessment questionnaire and attestation (confirm cadence with acquirer/Stripe).
- **Keep the Stripe SDK current** — deprecated/direct-post integrations can change SAQ eligibility.
- **Rotate keys** on any suspected exposure; treat any committed key as compromised and rotate immediately.
- **Never log** request bodies for payment routes at a verbosity that could capture tokens; payment logs record ids and outcomes only.
- **Access control** to the Stripe dashboard: least privilege, MFA on all Stripe accounts.

---

## 7. Non-negotiables (PCI)

- Queup never sees, stores, transmits, or logs a card number or CVV.
- Payment input is always the Stripe Payment Sheet.
- Webhook signatures are always validated; no skip path exists.
- Stripe secret + webhook keys are backend-only, never committed, never in a client bundle.
- Any change that would route card data through Queup is rejected — it breaks SAQ A.
