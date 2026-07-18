# Queup — Technical Architecture

**Workstream:** 04-technical
**Status:** Complete (v1 — MVP architecture)
**Depends on:** `00-master/README.md`, `00-master/claude-project-instructions.md`, `02-customer-journey/customer-app-flow.md`
**Feeds into:** `05-security/*`, `06-integrations/*`, `10-implementation/sprint-plan.md`

---

## 1. Purpose and scope

This document defines the end-to-end technical architecture for Queup: the three client applications, the backend API, the data layer, the third-party integrations, and the cross-cutting concerns (auth, location, payments, realtime, observability). It is the reference every other technical and security document builds on.

It describes the **MVP architecture** — the smallest system that delivers the confirmed customer, vendor, and manager journeys — and flags where the design leaves room for Phase 2 (reviews, subscriptions, multi-region).

---

## 2. System context

```
                         ┌──────────────────────────────────────────┐
                         │                 CLIENTS                    │
                         │                                            │
   ┌─────────────┐       │  ┌──────────────┐   ┌──────────────┐      │
   │  Customer   │───────┼─▶│ Customer App │   │  Vendor App  │      │
   │ (iOS/Android)│      │  │ RN + Expo    │   │  RN + Expo   │      │
   └─────────────┘       │  └──────┬───────┘   └──────┬───────┘      │
                         │         │                   │             │
   ┌─────────────┐       │         │          ┌────────┴───────┐     │
   │  Manager /  │───────┼────────────────────▶│  Manager Web   │     │
   │  Organiser  │       │         │          │  React + Vite  │     │
   └─────────────┘       │         │          └────────┬───────┘     │
                         └─────────┼───────────────────┼─────────────┘
                                   │  HTTPS (JWT)       │
                                   ▼                    ▼
                         ┌──────────────────────────────────────────┐
                         │        BACKEND API — Node + Fastify        │
                         │  routes → services → db/client             │
                         │  auth mw · rate-limit mw · validation      │
                         └───┬───────┬────────┬────────┬────────┬────┘
                             │       │        │        │        │
              ┌──────────────┘   ┌───┘    ┌───┘    ┌───┘    ┌───┘
              ▼                  ▼        ▼        ▼        ▼
      ┌───────────────┐  ┌───────────┐ ┌──────┐ ┌───────┐ ┌──────────┐
      │   Supabase    │  │  Stripe   │ │ FCM  │ │ what3 │ │  Sentry  │
      │ Postgres+     │  │  + Connect│ │(push)│ │ words │ │ (errors) │
      │ PostGIS·Auth· │  └───────────┘ └──────┘ └───────┘ └──────────┘
      │ Realtime      │
      └───────────────┘
```

**Trust boundary:** every client → backend call crosses the public internet and carries a Supabase-issued JWT. The backend is the only tier that holds service-role credentials and third-party secret keys. No client ever holds a Stripe secret key, the what3words key, or the Supabase service key.

---

## 3. Component inventory

### 3.1 Clients

| Component | Stack | Responsibility |
|---|---|---|
| **Customer app** | React Native + Expo (SDK 51+), TypeScript | Discovery (map + list), ordering, Stripe payment sheet, order tracking, push registration |
| **Vendor app** | React Native + Expo, TypeScript | Onboarding, Stripe Connect, menu builder, Go Live (Segment A GPS broadcast), order queue (realtime), payouts |
| **Manager web** | React + Vite, TypeScript | Vendor approval/suspension, order overview, event/geofence config, analytics, refunds & disputes |

### 3.2 Backend (`backend/`)

| Layer | Files | Responsibility |
|---|---|---|
| Entry | `src/index.ts` | Fastify bootstrap, plugin registration, graceful shutdown |
| Routes | `src/routes/{vendors,orders,payments,location,auth,notifications}.ts` | HTTP surface, request validation, response shaping |
| Services | `src/services/{stripe,notifications,w3w,geosearch}.service.ts` | Third-party calls and domain logic, transport-neutral |
| Middleware | `src/middleware/{auth,rateLimit,errorHandler}.ts` | JWT verification, per-route rate limits, uniform errors |
| Data | `src/db/{client.ts,migrations/,seeds/}` | Supabase singleton, SQL migrations, seed fixtures |
| Config | `src/config/env.ts` | Zod-validated environment loading (fail-fast on boot) |

### 3.3 Shared (`shared/`)

- `shared/types/index.ts` — `Vendor`, `MenuItem`, `Order`, `OrderItem`, `Customer`, `OrderStatus`, `LocationUpdate`, `VendorMode`, `LocationType`. Imported everywhere as `@queup/shared`.
- `shared/utils/` — `formatPrice(pence)`, `generateCollectionCode()`, `metresToDisplayDistance()`.

Sharing types across all four packages is the reason the monorepo exists: a change to the `Order` shape is a compile error in every app that consumes it.

---

## 4. Backend architecture

### 4.1 Layered design

```
HTTP request
   │
   ▼
[ Fastify route ]  ── validates input (Zod schema), extracts auth context
   │
   ▼
[ Service ]        ── domain logic + third-party orchestration, no HTTP knowledge
   │
   ▼
[ db/client ]      ── Supabase queries (parameterised), RPC calls
   │
   ▼
[ Supabase / Postgres ]
```

**Rule:** routes never talk to Stripe/w3w/FCM directly, and services never read `request`/`reply`. This keeps services reusable if the API is later re-exposed over a different transport, and keeps route handlers thin and testable.

### 4.2 Request lifecycle

1. **TLS termination** at the platform edge (Fly.io / Render / Railway — see §9).
2. **Rate-limit middleware** — token bucket keyed by IP + route class (see `05-security`).
3. **Auth middleware** — verifies the `Authorization: Bearer <jwt>`, attaches `request.auth = { userId, role, vendorId? }`. Public routes opt out explicitly.
4. **Input validation** — every body/query/param validated by a Zod schema at the route boundary. Invalid input → `400` with a machine-readable error code, never a stack trace.
5. **Handler** → service → db.
6. **Error handler** — maps thrown domain errors to HTTP codes; unexpected errors → `500` + Sentry capture, never leaking internals to the client.

### 4.3 Idempotency

- **Order creation** and **payment intent creation** accept an `Idempotency-Key` header. The key is passed straight through to Stripe for payment intents, and used to dedupe order rows (unique constraint on `(customer_id, idempotency_key)`).
- **Stripe webhooks** are idempotent by `event.id`: every processed event id is stored in `payment_events`; a replayed id is acknowledged with `200` and skipped.

---

## 5. Data architecture

### 5.1 Store

Single Postgres database on Supabase, with the **PostGIS** extension enabled (required before any migration runs). Supabase also provides Auth (GoTrue) and Realtime (logical replication → websockets) on the same database.

### 5.2 Core entities (see `database-schema.md` for full DDL)

`customers` · `vendors` · `menu_items` · `orders` · `order_items` · `payment_events` · `collection_slots` (Segment B) · `devices` (push tokens) · `events` (manager geofences).

### 5.3 The location model — the defining architectural rule

Location is the single most important domain rule in Queup, and it forks the entire system by vendor segment.

| | **Segment A** (`festival`, `fixed_pitch`, `popup`) | **Segment B** (`home_food`) |
|---|---|---|
| `vendors.location_type` | `'w3w'` | `'postal'` |
| Populated column | `w3w_address` + transient `geom` (current point) | `postal_address` JSONB (`line1`, `line2`, `city`, `postcode`) + `postcode_centroid` geom |
| Source of truth | Live GPS from vendor app, every 30s | Fixed address set once at onboarding |
| Persistence | **Current position only** — overwritten each update, never a history table | **Permanent** — treated as personal data |
| what3words | One API call per location update | **Never called** |
| Customer proximity | PostGIS `ST_DWithin` on `geom` (metres) | Postcode-centroid distance (miles) |
| Customer app UI | w3w badge + "Take me there" | Full street address + postcode + distance in miles |
| Vendor app | Background location active | Background location **not requested** |

This rule is enforced in three places and must stay consistent across all of them: the database (`location_type` + nullable columns + a `CHECK` constraint), the `location` route (rejects GPS updates from `postal` vendors), and `geosearch.service.ts` (branches on `location_type`).

### 5.4 Geospatial search

Segment A discovery is a PostGIS query wrapped in an RPC, `vendors_within_radius(lat, lng, radius_m)`, using a GiST index on `vendors.geom`. Default radius 500 m. Segment B discovery is an ordinary indexed query ordered by distance from the customer's postcode centroid — no PostGIS radius scan, no realtime GPS.

---

## 6. Key runtime flows

### 6.1 Segment A location broadcast (vendor → customer)

```
Vendor app (Go Live, Segment A only)
  every 30s: expo-location → { lat, lng }
     │  POST /location/update  (JWT, vendor role)
     ▼
Backend location route
  ├─ reject if vendor.location_type != 'w3w'
  ├─ w3w.service.convertToWords(lat,lng)   ← ONE call per update
  └─ UPDATE vendors SET geom=Point, w3w_address=... WHERE id=vendorId
     │
     ▼
Customer app discovery
  POST /vendors/nearby { lat, lng, radius }
  → vendors_within_radius RPC → cards with w3w badge + distance
```

The 30-second cadence and the one-call-per-update rule together bound what3words usage to well inside the 25k/month free tier at MVP scale, and are both non-negotiable (see `CLAUDE.md` "YOU MUST").

### 6.2 Order + payment (customer)

```
Customer basket → Checkout
  POST /orders            (Idempotency-Key) → order row (status=pending), server-computed totals
  POST /payments/intent   → Stripe PaymentIntent with:
                              amount = order total (pence)
                              application_fee_amount = 5% platform fee
                              transfer_data.destination = vendor Connect account
  → client confirms with Stripe Payment Sheet (card data never touches Queup)
  → Stripe → POST /payments/webhook (signature verified)
       payment_intent.succeeded → order.status stays pending, payment_confirmed=true
                                 → generate collection code (server-side)
                                 → FCM push to vendor: "New order"
```

Totals are **always recomputed server-side** from current menu prices; the client-supplied total is never trusted. All money is integer pence.

### 6.3 Order fulfilment (vendor)

```
Vendor app Order Queue  ── Supabase Realtime subscription on orders WHERE vendor_id=me
  new order appears live (no polling)
  Accept   → PATCH /orders/:id/status { accepted }
  Prepare  → { preparing }
  Ready    → { ready }  → FCM push to customer: "Your order is ready — code ABCD"
  Collected→ { collected } (terminal)
```

Status transitions are validated server-side against the allowed one-directional graph; an illegal transition (e.g. `ready → pending`) is rejected `409`. Rejection is the only non-forward move: `pending → rejected` triggers a Stripe refund.

### 6.4 Collection codes

Generated server-side only, 4 characters, alphanumeric uppercase, excluding visually ambiguous `0/O/1/I`. Unique per active order per vendor (a short code can be reused once an order is `collected`). The customer shows the code; the vendor matches it in the order detail screen.

---

## 7. Realtime & notifications

- **Realtime (Supabase):** powers the vendor Order Queue and the customer Order Status Tracker. Both subscribe to filtered `orders` changes over websockets. Row-Level Security applies to realtime too — a vendor only receives their own orders, a customer only their own.
- **Push (FCM):** transactional events (new order → vendor; order ready → customer; Segment B slot reminder 2h before). Device tokens live in `devices`, registered at app start and refreshed on rotation. FCM server key is backend-only.

Realtime is for *in-app live state while the screen is open*; push is for *reaching a user whose app is backgrounded*. Both fire for the same events so neither is a single point of failure.

---

## 8. Cross-cutting concerns

| Concern | Approach |
|---|---|
| **Auth** | Supabase Auth issues JWTs; backend verifies signature + role claim. Full design in `05-security/auth-design.md`. |
| **Authorization** | Role claim (`customer`/`vendor`/`admin`) checked in middleware; Postgres RLS as defence-in-depth. |
| **Validation** | Zod at every route boundary; DB `CHECK`/`FK`/`NOT NULL` as the backstop. |
| **Config** | `config/env.ts` validates all env vars on boot; process exits if any required secret is missing. |
| **Errors** | Central Fastify error handler; domain errors carry a `code`; unexpected → Sentry + generic `500`. |
| **Observability** | Sentry (client + backend), structured JSON logs (pino), health check at `/healthz`. |
| **Money** | Integer pence everywhere; `formatPrice()` only at the display edge. |
| **Time** | UTC in the database; localise to Europe/London at the display edge. |

---

## 9. Deployment & environments

| Environment | Backend | Database | Clients |
|---|---|---|---|
| **dev** | local Fastify `:3000` | Supabase dev project | Expo Go / dev client, Vite dev server |
| **staging** | container on host (Fly.io/Render) | Supabase staging project | Expo internal distribution, staging web |
| **production** | container, autoscaled | Supabase prod project (paid tier) | App Store / Play Store, hosted web |

- **Backend** ships as a container image; TypeScript compiled ahead of time (`npm run build`), run as plain Node. Stateless — horizontal scale behind the platform load balancer.
- **Mobile** built with EAS Build; over-the-air JS updates via Expo Updates (signed). Native changes require a store submission.
- **Manager web** static build served from the edge/CDN.
- **Secrets** injected as environment variables per environment — never in the repo. `.env` is git-ignored and documented by `.env.example` only.
- **Migrations** run as a gated deploy step against the target Supabase project; PostGIS enabled first.

---

## 10. Monorepo layout

```
queup/
├── apps/
│   ├── customer-app/   src/{screens,hooks,components,services}
│   ├── vendor-app/     src/{screens,hooks,services}
│   └── manager-web/    src/{pages,components,hooks}
├── backend/            src/{index.ts,routes,services,middleware,db,config}
├── shared/             types/index.ts, utils/
└── docs/               all workstream documentation (markdown)
```

npm workspaces link `@queup/shared` into every package. See `code-standards.md` for conventions and `claude-code-instructions.md` for the Claude Code build workflow.

---

## 11. Architectural risks & decisions

| Risk / decision | Position |
|---|---|
| **RPC/method drift** — n/a here (not the NotebookLM protocol) | Queup owns its own API surface; no undocumented third-party RPC IDs. |
| **what3words rate limit** | Bounded by 30s cadence + one-call-per-update; monitor usage, alert at 80% of free tier. |
| **Vendor location privacy** | Segment A GPS is transient (overwritten, never historised) — a deliberate data-minimisation choice, see `05-security/gdpr-compliance.md`. |
| **Stripe as fund holder** | Queup never holds customer funds → lighter FCA and PCI posture (SAQ A). |
| **Single region (UK)** | MVP is UK-only; Europe/London + GBP assumptions are baked in and documented, not hidden. |
| **Supabase free tier limits** | 500 MB / 50k MAU is fine for MVP; the transient-location design keeps row churn off disk growth. Paid tier at scale. |

---

## 12. Cross-references

- Data model & DDL → `04-technical/database-schema.md`
- HTTP contract → `04-technical/api-reference.md`
- Conventions → `04-technical/code-standards.md`
- Build workflow → `04-technical/claude-code-instructions.md`
- Auth, GDPR, PCI, checklist → `05-security/*`
- Integration guides → `06-integrations/*`
