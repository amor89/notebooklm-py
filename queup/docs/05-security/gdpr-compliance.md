# Queup — UK GDPR Compliance

**Workstream:** 05-security
**Status:** Complete (v1 — engineering & policy basis; not legal advice)
**Depends on:** `04-technical/database-schema.md` (§11 data classification)
**Feeds into:** `07-legal/privacy-policy-draft.md`, `07-legal/vendor-agreement.md`
**Regime:** UK GDPR + Data Protection Act 2018 (post-Brexit). Regulator: **ICO** (ico.org.uk).

> This document defines how Queup handles personal data at the engineering and policy level. It is the input to the solicitor-reviewed privacy policy in workstream 07 — it is **not** a substitute for legal advice.

---

## 1. Roles under UK GDPR

- **Queup is the data controller** for customer accounts, vendor accounts, and orders placed through the platform.
- **Vendors are independent controllers** for their own food-business obligations (allergens, food safety) and become joint context only for order fulfilment. Queup is **not** a food business (FSA registration is the vendor's responsibility).
- **Processors engaged by Queup** (Article 28 data-processing agreements required): **Supabase** (hosting/DB/auth), **Stripe** (payments), **Firebase/Google** (push), **what3words** (location encoding), **Sentry** (error monitoring).

Each processor must have a signed DPA on file before production launch, and each must be listed in the privacy policy as a sub-processor.

---

## 2. Lawful bases (Article 6)

| Processing | Lawful basis |
|---|---|
| Creating & operating a customer/vendor account | **Contract** (Art. 6(1)(b)) |
| Taking payment, splitting to vendor via Stripe | **Contract** |
| Sending order/status push notifications | **Contract** (transactional) |
| Retaining order & payment records for tax/accounting | **Legal obligation** (Art. 6(1)(c)) |
| Real-time proximity search using customer location | **Consent** (device permission) + legitimate interest, location **not stored** |
| Marketing emails/notifications | **Consent** (explicit opt-in; `marketing_opt_in`) |
| Fraud prevention & platform security | **Legitimate interest** (Art. 6(1)(f)) |

No special-category data is processed. Age is confirmed by self-declaration (18+, or 13+ with parental consent) at sign-up, not by collecting date-of-birth.

---

## 3. Data map (what we hold, where, why, how long)

Derived directly from `04-technical/database-schema.md` §11.

| Data | Subject | Store | Basis | Retention |
|---|---|---|---|---|
| Name, email, phone | Customer / vendor | Supabase (`customers`, `auth.users`) | Contract | Life of account, then erased |
| Password hash | Customer / vendor | Supabase Auth (hashed, salted) | Contract | Life of account |
| `home_postcode` (coarse, optional) | Customer | Supabase | Consent | Life of account |
| **Live customer GPS** | Customer | **Never stored** — transient, in-request only | Consent | **n/a** |
| **Segment A vendor GPS / w3w** | Vendor | Supabase `vendors.geom/w3w_address` — **current point only, overwritten each update** | Contract | Not historised; cleared on go-offline / deletion |
| **Segment B vendor postal address** | Vendor | Supabase `vendors.postal_address` — **permanent, personal data** | Contract | Life of vendor account |
| Order & item history | Customer / vendor | Supabase `orders`, `order_items` | Contract + legal obligation | **6 years** (tax) then anonymised |
| Payment records (no card data) | Customer / vendor | Stripe (source of truth) + `payment_events` (no PAN) | Legal obligation | Per Stripe + 6 yrs reconciliation |
| **Card data (PAN/CVV)** | Customer | **Stripe only — never touches Queup** | — | n/a (see PCI notes) |
| Push device token | Any user | Supabase `devices` | Contract | Until sign-out / rotation / deletion |
| Error telemetry | Any user | Sentry (PII scrubbed) | Legitimate interest | 90 days |

### The location minimisation stance (data-protection by design)

- **Customer location is never persisted.** It arrives in a discovery request, drives a PostGIS/postcode query, and is discarded. No location-history table exists for customers.
- **Segment A vendor location is transient.** Each 30-second update overwrites the single current point — there is deliberately no movement-history table. The privacy policy states the live position is broadcast but not retained.
- **Segment B vendor location is a fixed postal address**, stored permanently and treated as personal data, covered by the erasure right.

This asymmetry (§`04-technical/database-schema.md`) is a purpose-built Article 5(1)(c) **data-minimisation** control, not an accident of implementation.

---

## 4. Data subject rights (and how each is served)

| Right (UK GDPR) | Queup implementation |
|---|---|
| **Access** (Art. 15) | Self-service export of profile + order history from the account screen; manual DSAR handling within one month via privacy@queup.app |
| **Rectification** (Art. 16) | Users edit profile/address in-app |
| **Erasure** (Art. 17) | In-app "Delete my account" → erasure cascade (§5) |
| **Restriction** (Art. 18) | Account suspension flag; processing limited to legal-obligation retention |
| **Portability** (Art. 20) | Export delivered as machine-readable JSON |
| **Object** (Art. 21) | Marketing opt-out toggle; object to legitimate-interest processing via DSAR |
| **Withdraw consent** (Art. 7) | Revoke location permission (OS); marketing opt-out; consequences explained |

DSAR SLA: acknowledge promptly, fulfil within **one calendar month** (extendable by two months for complexity, with notice).

---

## 5. Right to erasure — cascade design

Account deletion must remove personal data while preserving records the law requires Queup to keep (tax/accounting) in **anonymised** form.

```
Delete account (customer)
  ├─ auth.users row deleted → customers row cascades (name, email, phone, postcode gone)
  ├─ devices rows cascade (push tokens gone)
  ├─ orders: NOT hard-deleted → customer_id retained for accounting but
  │     personal fields anonymised (name snapshots, special_instructions cleared),
  │     linked to a tombstone "deleted user" — kept 6 yrs then purged
  └─ Stripe customer object deleted via API (card tokens removed at Stripe)

Delete account (vendor)
  ├─ vendors row + menu_items cascade
  ├─ Segment B postal_address erased; Segment A geom already transient
  ├─ orders anonymised as above (both sides)
  └─ Stripe Connect account deactivated (payout/KYC records retained by Stripe per law)
```

- **Why anonymise, not delete, orders:** HMRC record-keeping (≈6 years) is a legal obligation that overrides erasure for the transactional record — but personal identifiers within it are stripped.
- Erasure is **irreversible** and confirmed to the user; active in-flight orders must be completed or refunded before deletion proceeds.

---

## 6. Consent & transparency at sign-up

- Privacy policy presented and **explicitly accepted** at sign-up (not pre-ticked).
- Separate, unbundled **marketing opt-in** (default off).
- Location permission requested with a clear purpose string ("show vendors near you; we never share your location").
- Age gate confirmed at sign-up.
- Layered notice: short in-app summary + link to the full policy.

---

## 7. International transfers

Processors may process data outside the UK (e.g. US-based Stripe/Firebase/Sentry). Each transfer relies on the processor's **UK IDTA / SCCs / UK-US Data Bridge** as applicable, documented in the sub-processor list. Prefer EU/UK data regions where a processor offers them (e.g. Supabase UK/EU region).

---

## 8. Security of processing (Article 32)

Cross-references `auth-design.md` and `security-checklist.md`:
- Encryption in transit (TLS) and at rest (Supabase/Stripe managed).
- Access control: least privilege, role claims, RLS, admin MFA.
- Pseudonymisation/minimisation: transient location, no card data, PII-scrubbed telemetry.
- Resilience: managed backups (Supabase), incident response (§9).
- Regular review: security checklist gates each release.

---

## 9. Breach response

- **Detection → assessment → containment → notification.**
- Personal-data breach likely to risk individuals' rights: **notify the ICO within 72 hours**; notify affected individuals without undue delay where high risk.
- Maintain an internal breach register regardless of reportability.
- Processors (Supabase/Stripe/etc.) are contractually required to notify Queup of breaches on their side without undue delay (Art. 28 DPAs).

---

## 10. Accountability & governance

- **ICO registration / data-protection fee:** register with the ICO before processing personal data at launch.
- **Records of processing (ROPA):** maintain this data map as the Article 30 record.
- **DPIA:** conduct a Data Protection Impact Assessment for the location-processing feature (systematic monitoring of location) before launch — the transient-storage design is its primary mitigation.
- **Privacy by design & default:** minimise collection, default marketing off, default to coarse location, no card data.
- **DPO:** a formal DPO is likely not mandatory at MVP scale, but assign a named privacy owner and the privacy@queup.app contact.

---

## 11. Handover to legal (workstream 07)

This document supplies the privacy-policy draft with: the data map (§3), lawful bases (§2), retention periods (§3/§5), sub-processor list (§1/§7), the location-storage commitments (§3), erasure behaviour (§5), and breach/rights processes. The policy must state plainly:
- Segment A vendor GPS is broadcast live but **not stored** long-term.
- Segment B vendor postal address **is stored** permanently.
- Customer GPS is **never stored**.
- Right to erasure applies to both vendor types, with the anonymised-order carve-out for legal retention.
