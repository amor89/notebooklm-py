# Queup — Authentication & Authorization Design

**Workstream:** 05-security
**Status:** Complete (v1)
**Depends on:** `04-technical/architecture.md`, `04-technical/database-schema.md`, `04-technical/api-reference.md`
**Feeds into:** `05-security/security-checklist.md`, `07-legal/privacy-policy-draft.md`

---

## 1. Model overview

Queup uses **Supabase Auth (GoTrue)** as the identity provider. Supabase issues signed **JWTs**; the Queup backend is a **resource server** that verifies those tokens and enforces authorization. There is no custom password store, no custom token minting — Supabase owns credentials, hashing, refresh, and OAuth.

```
Client ──sign in──▶ Supabase Auth ──issues──▶ JWT (access + refresh)
Client ──Bearer JWT──▶ Queup backend ──verify sig + claims──▶ route + RLS
```

**Three principals, one identity source:** `customer`, `vendor`, `admin`. Every JWT carries a `role` claim; the backend and Postgres RLS both key off it.

---

## 2. Authentication methods

| Method | Availability | Notes |
|---|---|---|
| Email + password | All apps | Min 8 chars, ≥1 number (client + server enforced). Verification email required before first use. |
| Continue with Google | Customer + vendor | Supabase OAuth provider |
| Continue with Apple | Customer + vendor (iOS) | Supabase OAuth provider; required by App Store when other social login is offered |
| Manager (admin) | Manager web only | Email+password with **mandatory MFA** (TOTP); no self-service admin sign-up |

- **Email verification** is required before a customer can order or a vendor can go live.
- **Password reset** is Supabase's email-link flow; tokens are single-use and short-lived.
- **Age gate:** sign-up requires `age_confirmed = true` (18+, or 13+ with parental consent). Recorded on the `customers` row and surfaced in the privacy policy.

---

## 3. Tokens

| Token | Lifetime | Storage (client) |
|---|---|---|
| Access JWT | ~1 hour | In memory; **never** in `localStorage` on web. Mobile: Expo SecureStore (Keychain / Keystore). |
| Refresh token | Long-lived, rotating | Expo SecureStore (mobile) / httpOnly-equivalent secure storage (web). Rotated on each use. |

- **Rotation:** refresh tokens rotate on redemption; a reused (stolen) refresh token invalidates the session family — Supabase default, kept on.
- **Signature:** the backend verifies the JWT signature against the Supabase project JWKS (asymmetric) or shared secret, plus `exp`, `iss`, and `aud`. An expired or malformed token → `401 unauthenticated`.
- **No secrets client-side:** clients hold only the Supabase **anon** key and Stripe **publishable** key. The Supabase **service key**, Stripe **secret** key, and w3w key are backend-only.

---

## 4. The `role` claim & authorization

The `role` claim is set from `auth.users` app metadata at issuance and is **not** user-editable. Authorization is enforced in two layers (defence-in-depth):

### 4.1 Backend middleware (`auth.middleware.ts`)

```
verify signature + exp/iss/aud
  → attach request.auth = { userId, role, vendorId? }
  → route declares required role(s); mismatch → 403 forbidden
  → ownership checks (e.g. order.vendor_id belongs to request.auth.vendorId)
```

- Public routes opt out explicitly (discovery, `/healthz`, Stripe webhook).
- Vendor routes require `role = vendor` **and** ownership of the target vendor.
- Admin routes require `role = admin`.

### 4.2 Postgres Row-Level Security (backstop)

Even if a route check were missed, RLS on every user-data table restricts rows to the authenticated `auth.uid()` / role (see `database-schema.md` §10). RLS also governs **Supabase Realtime**, so the vendor Order Queue and customer Order Tracker only stream rows the subscriber is allowed to see.

### 4.3 Route → role matrix (summary)

| Route class | customer | vendor | admin | public |
|---|:-:|:-:|:-:|:-:|
| `/vendors/nearby`, `/vendors/home-food`, `GET /vendors/:id`, `/vendors/:id/menu` | ✓ | ✓ | ✓ | ✓ |
| `POST/PATCH /vendors`, `/menu`, `/location/*`, `/payments/connect/*`, `GET /payments/payouts` | | ✓ (owner) | | |
| `POST /orders`, `POST /payments/intent`, `GET /orders` (own) | ✓ | | | |
| `PATCH /orders/:id/status`, `/orders/:id/verify-code` | | ✓ (owner) | | |
| `/admin/*` | | | ✓ | |
| `/payments/webhook` | | | | ✓ (Stripe-signed) |
| `/healthz` | | | | ✓ |

Full contract in `04-technical/api-reference.md`.

---

## 5. Session & device management

- **Push device binding:** `devices` maps `user_id → fcm_token`; tokens are refreshed on rotation and removed on sign-out and on account deletion.
- **Sign-out** revokes the refresh token server-side and clears SecureStore.
- **Multiple devices** per user are supported; each has its own refresh-token family.
- **Manager MFA:** admin sessions require a TOTP second factor at login; a lost factor is recovered via a manual, identity-verified process (not self-service).

---

## 6. Vendor onboarding & Stripe identity

- A vendor account is a `vendor`-role user + a `vendors` row starting `status = pending_review`.
- **Stripe Connect identity verification** is separate from Queup auth: a vendor authenticates to Queup, then completes Stripe's KYC via a Connect account link. Payouts are blocked until `stripe_onboarded = true` (webhook `account.updated`).
- A manager approves the vendor (`pending_review → active`) before they appear in discovery.

---

## 7. Threats considered

| Threat | Mitigation |
|---|---|
| Credential stuffing / brute force | Rate limit `/auth/*` (10/min/IP); Supabase lockout; strong password policy |
| Token theft | Short access-token TTL; SecureStore only; refresh rotation with reuse detection |
| Privilege escalation | `role` is server-set metadata, not client-editable; RLS backstop |
| IDOR (accessing others' orders/vendors) | Ownership checks in middleware **and** RLS row filters |
| Session fixation | Supabase rotates tokens; no app-managed session ids |
| OAuth token misuse | Providers configured server-side; redirect URIs allow-listed |
| Admin account compromise | Mandatory MFA; no admin self-sign-up; audit of admin actions |
| Missing auth on a new route | Deny-by-default: routes require an explicit `public: true` to skip auth |

---

## 8. Non-negotiables (auth)

- Deny by default — a route without an explicit public flag requires a valid JWT.
- `role` claim is authoritative and server-controlled; never trust a client-sent role.
- Access tokens never in web `localStorage`; mobile tokens in SecureStore only.
- Admin access always MFA-gated.
- RLS enabled on every user-data table, including for Realtime.
- All of the above verified in `security-checklist.md` before launch.
