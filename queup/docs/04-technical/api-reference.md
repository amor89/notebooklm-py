# Queup — API Reference

**Workstream:** 04-technical
**Status:** Complete (v1 — MVP surface)
**Base URL:** `https://api.queup.app/v1` (prod) · `http://localhost:3000/v1` (dev)
**Transport:** Fastify · JSON · TLS only

---

## 1. Conventions

- **Auth:** `Authorization: Bearer <supabase-jwt>` on every route except those marked **Public**. See `05-security/auth-design.md`.
- **Roles:** each route lists the required role claim — `customer`, `vendor`, or `admin`.
- **Money:** all amounts are integer **pence**.
- **IDs:** UUID v4.
- **Timestamps:** ISO-8601 UTC (`2026-07-18T14:30:00Z`).
- **Validation:** every request body/query is validated by a Zod schema at the route boundary; failures return `400` with `code: "validation_error"` and a `fields` map. No stack traces leak.
- **Idempotency:** `Idempotency-Key: <uuid>` accepted on `POST /orders` and `POST /payments/intent`.
- **Errors:** uniform envelope:

```json
{ "error": { "code": "order_not_found", "message": "No order with that id", "status": 404 } }
```

| Code | HTTP | Meaning |
|---|---|---|
| `validation_error` | 400 | Body/query failed schema validation |
| `unauthenticated` | 401 | Missing/invalid JWT |
| `forbidden` | 403 | Valid JWT, wrong role/ownership |
| `not_found` | 404 | Resource does not exist or not visible |
| `conflict` | 409 | Illegal state transition / duplicate |
| `rate_limited` | 429 | Rate limit exceeded (`Retry-After` header) |
| `internal` | 500 | Unexpected — captured in Sentry, generic message |

---

## 2. Auth & profile

### `POST /auth/register` — **Public**
Thin wrapper over Supabase sign-up that also creates the `customers` profile row and records the age gate.

```jsonc
// request
{ "email": "a@b.com", "password": "…", "full_name": "Amy R", "age_confirmed": true, "marketing_opt_in": false }
// 201
{ "user_id": "…", "verification_required": true }
```
`age_confirmed` must be `true` or the request is rejected `400`.

### `GET /me` — customer | vendor | admin
Returns the caller's profile derived from the JWT (`customers` or `vendors` row + role).

### `POST /devices` — any authenticated role
Register/refresh an FCM token.
```jsonc
{ "fcm_token": "…", "platform": "ios" } // 204
```

---

## 3. Vendors & discovery

### `POST /vendors/nearby` — **Public** (customer discovery, Segment A)
PostGIS radius search over live `w3w` vendors.
```jsonc
// request
{ "lat": 51.4545, "lng": -2.5879, "radius_m": 500 }
// 200
{ "vendors": [
  { "id": "…", "business_name": "Smokehouse", "category": "bbq",
    "w3w_address": "///filled.count.soap", "distance_m": 143.2,
    "estimated_wait_min": 12, "dietary_tags": ["halal"], "last_seen_at": "…" }
]}
```
`radius_m` ∈ {250, 500, 1000, 2000}; default 500. Never returns Segment B vendors.

### `GET /vendors/home-food` — **Public** (customer discovery, Segment B)
Postcode-centroid distance list. No GPS, no what3words.
```jsonc
// query: ?postcode=BS1+4DJ&category=bakery
// 200
{ "vendors": [
  { "id": "…", "business_name": "Rosa's Bakes", "speciality": "bakery",
    "postal_address": { "line1": "12 Mill Rd", "city": "Bristol", "postcode": "BS1 4DJ" },
    "distance_miles": 1.4, "next_slot": "2026-07-20T10:00:00Z", "price_range": "£" }
]}
```

### `GET /vendors/:id` — **Public**
Full vendor profile + menu. Response includes a `location` block that differs by mode:
```jsonc
// Segment A
"location": { "type": "w3w", "w3w_address": "///filled.count.soap", "estimated_wait_min": 12 }
// Segment B
"location": { "type": "postal", "postal_address": { "line1": "…", "postcode": "…" }, "slots": [ … ] }
```

### `POST /vendors` — vendor
Create the caller's vendor. `vendor_mode` fixes `location_type` server-side; supplying a mismatched pair is rejected `400`. Starts in `status: pending_review`.

### `PATCH /vendors/:id` — vendor (owner only)
Update profile/menu metadata. Ownership enforced; `403` otherwise.

---

## 4. Menu

### `GET /vendors/:id/menu` — **Public**
### `POST /vendors/:id/menu` — vendor (owner)
### `PATCH /menu/:itemId` — vendor (owner) — includes `is_available` toggle
### `DELETE /menu/:itemId` — vendor (owner)

`price_pence` is a non-negative integer; floats rejected `400`.

---

## 5. Location (Segment A only)

### `POST /location/update` — vendor (owner, `location_type = 'w3w'`)
Called by the vendor app every 30 s while "Live".
```jsonc
// request
{ "lat": 51.4545, "lng": -2.5879 }
// 200
{ "w3w_address": "///filled.count.soap", "last_seen_at": "2026-07-18T14:30:00Z" }
```
- Rejected `409` if the vendor's `location_type = 'postal'` (home food never broadcasts).
- Exactly **one** what3words API call per request (never more).
- Overwrites `geom`/`w3w_address`/`last_seen_at` — no history retained.
- Rate-limited to protect the w3w free tier (see §9).

### `POST /location/offline` — vendor
Clears live position (Go offline). Sets `geom = null`; vendor drops off the map.

---

## 6. Orders

### `POST /orders` — customer  · `Idempotency-Key`
Server recomputes totals from current menu prices and the 5% platform fee; client totals ignored.
```jsonc
// request
{ "vendor_id": "…", "items": [ { "menu_item_id": "…", "quantity": 2, "special_instructions": "no onions" } ],
  "slot_id": null /* required for Segment B */ }
// 201
{ "id": "…", "status": "pending", "subtotal_pence": 1700,
  "platform_fee_pence": 85, "total_pence": 1785, "collection_code": null }
```
`collection_code` stays `null` until payment succeeds. Duplicate `Idempotency-Key` returns the original order, not a new one.

### `GET /orders/:id` — customer (owner) | vendor (owner)
### `GET /orders?role=vendor&status=preparing` — vendor (own orders) | customer (own)

### `PATCH /orders/:id/status` — vendor (owner)
```jsonc
{ "status": "accepted" } // → 200 with new state
```
Validated against the one-directional graph; illegal transition → `409`. `pending → rejected` triggers a Stripe refund and a customer push.

### `POST /orders/:id/verify-code` — vendor (owner)
Vendor matches the customer's collection code at handover.
```jsonc
{ "collection_code": "K7QP" } // 200 { "match": true }
```

---

## 7. Payments (Stripe Connect)

### `POST /payments/connect/onboard` — vendor
Creates/returns a Stripe Connect account link for identity verification. Payouts blocked until `stripe_onboarded = true`.

### `POST /payments/intent` — customer · `Idempotency-Key`
```jsonc
// request
{ "order_id": "…" }
// 200
{ "client_secret": "pi_…_secret_…", "publishable_key": "pk_live_…" }
```
Server builds the PaymentIntent with `amount = order.total_pence`, `application_fee_amount = order.platform_fee_pence`, and `transfer_data.destination = vendor.stripe_account_id`. The client confirms with the Stripe Payment Sheet — **card data never reaches Queup** (SAQ A; see `05-security/pci-dss-notes.md`).

### `POST /payments/webhook` — **Public (Stripe-signed)**
Signature verified with `stripe.webhooks.constructEvent` using `STRIPE_WEBHOOK_SECRET` — **never skipped**. Idempotent by `event.id` via `payment_events`.
Handled events:
| Event | Action |
|---|---|
| `payment_intent.succeeded` | mark order `payment_confirmed`, generate collection code, push vendor |
| `payment_intent.payment_failed` | mark order for retry, notify customer |
| `charge.refunded` | reconcile refund on `rejected`/support refund |
| `account.updated` | update `stripe_onboarded` |

Unrecognised or replayed events → `200` (acknowledged, skipped).

### `GET /payments/payouts` — vendor
Summary of Connect transfers/balance for the payouts screen.

---

## 8. Manager (admin)

All routes require the `admin` role claim.

| Method & path | Purpose |
|---|---|
| `GET /admin/vendors?status=pending_review` | Vendor approval queue |
| `PATCH /admin/vendors/:id/status` | `active` / `suspended` |
| `GET /admin/orders` | Platform-wide order overview (filters, pagination) |
| `POST /admin/events` | Create geofenced event zone (`boundary` polygon) |
| `GET /admin/analytics` | Revenue, order counts, active-vendor metrics |
| `POST /admin/orders/:id/refund` | Support-initiated refund (Stripe) |

Pagination: `?limit=` (default 20, max 100) + `?cursor=`.

---

## 9. Rate limiting

Token-bucket per IP + route class (see `05-security/security-checklist.md`):

| Route class | Limit |
|---|---|
| Auth (`/auth/*`) | 10 / min / IP |
| Discovery (`/vendors/nearby`, `/vendors/home-food`) | 60 / min / IP |
| Location (`/location/update`) | 3 / min / vendor (guards 30 s cadence + w3w quota) |
| Orders / payments | 30 / min / user |
| Default | 100 / min / IP |

Exceeded → `429` + `Retry-After`.

---

## 10. Health & ops

- `GET /healthz` — **Public**, liveness (no auth, no DB write). Returns build SHA + `ok`.
- `GET /readyz` — readiness: checks DB + Stripe reachability.
