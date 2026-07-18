# Queup — Code Standards

**Workstream:** 04-technical
**Status:** Complete (v1)
**Applies to:** `backend/`, `apps/*`, `shared/`

These are the conventions every contributor and every Claude Code session follows. They exist to keep four packages (backend + three clients) coherent through a shared type layer.

---

## 1. Language & typing

- **TypeScript strict mode everywhere.** `"strict": true` in every `tsconfig.json`. No exceptions.
- **No `any`.** Use `unknown` + a narrowing guard, or a proper type. `// @ts-ignore` is banned; if a third-party type is wrong, write a `.d.ts` shim and comment why.
- **`tsc --noEmit` must pass** before every commit. This is a hard gate (pre-commit + CI).
- **Named exports only. No default exports.** Default exports break rename-safety and tree-shaking, and make grep/imports ambiguous.

```ts
// good
export function formatPrice(pence: number): string { … }
// banned
export default function (pence) { … }
```

---

## 2. Shared types are the contract

- All cross-package domain types live in `shared/types/index.ts` and are imported as `@queup/shared`:

```ts
import type { Order, Vendor, OrderStatus, VendorMode, LocationType } from "@queup/shared";
```

- A change to a shared type is a compile error in every consumer — that is the point. Never re-declare a domain shape locally to "avoid the import".
- Enum-like domains (`OrderStatus`, `VendorMode`, `LocationType`) are string-literal unions that mirror the DB `CHECK` constraints exactly. Keep the two in lockstep; a migration that adds a value updates the union in the same PR.

---

## 3. Money & units

- **All prices are integer pence.** `850` = £8.50. Never `float`, never `numeric`, never a decimal string in code.
- Arithmetic stays in pence; convert to a display string only at the UI edge via `formatPrice()`.
- Distances: metres internally; `metresToDisplayDistance()` decides metres-vs-miles for display (Segment A metres under 1 km then miles; Segment B always miles).
- Time: UTC everywhere; localise to `Europe/London` only at the display edge.

---

## 4. Backend structure

- **Layering is enforced by discipline:** `routes → services → db`. Routes never call Stripe/w3w/FCM directly; services never touch `request`/`reply`.
- **One Zod schema per route**, validating body/query/params at the boundary. The validated, typed object is what the handler passes down.
- **Config via `config/env.ts` only.** All env vars are read once, validated by Zod at boot, and exported typed. Reading `process.env` anywhere else is banned.
- **Errors:** throw typed domain errors (`new NotFoundError("order")`); the central error handler maps them to HTTP. Never `reply.send` a raw caught error.
- **Every mutating route is idempotent-safe** where it can be (see `api-reference.md` §Idempotency).

---

## 5. React Native / React

- **Functional components + hooks only.** No class components.
- Screen files own layout; data-fetching lives in hooks (`useNearbyVendors`, `useOrderStatus`, `useLocationBroadcast`).
- **Segment-aware UI is explicit, never implicit:** components branch on `vendor.location_type`, and the two vendor-card variants (w3w badge vs street address) are distinct components, not a pile of conditionals inside one.
- Background location (`expo-location`) is requested **only** in the vendor app and **only** for Segment A modes. The customer app never requests background location; home-food vendor screens never request it at all.
- No secret keys in any client bundle — ever (Stripe secret, w3w key, Supabase service key are backend-only). Clients hold only the Supabase anon key and Stripe publishable key.

---

## 6. Security-adjacent rules (see `05-security/*`)

- **Never commit `.env`.** It is git-ignored; `.env.example` documents the keys with placeholder values.
- **Always validate Stripe webhook signatures** with `stripe.webhooks.constructEvent`. No code path skips it.
- **Never call what3words for `home_food` vendors**, and never more than once per Segment A location update.
- **Never trust client-supplied totals** — recompute server-side from current menu prices.
- All user input is validated (Zod) and all DB access is parameterised (Supabase client) — no string-built SQL.

---

## 7. Naming & files

| Kind | Convention | Example |
|---|---|---|
| Files | kebab-case | `geosearch.service.ts` |
| Services | `*.service.ts` | `stripe.service.ts` |
| Middleware | `*.middleware.ts` | `auth.middleware.ts` |
| React components | PascalCase | `VendorCardW3W.tsx` |
| Hooks | `use*` | `useLocationBroadcast.ts` |
| Types | PascalCase | `OrderStatus` |
| DB tables/columns | snake_case | `collection_code` |
| Env vars | SCREAMING_SNAKE | `STRIPE_WEBHOOK_SECRET` |

---

## 8. Testing

| Tier | Location | Scope |
|---|---|---|
| Unit | `*/**.test.ts` next to source | Pure logic: `formatPrice`, code generation, status-graph guard, totals math |
| Integration | `backend/test/integration/` | Routes against a test Supabase project or a Postgres container |
| E2E (manual/device) | — | Location features **must** be tested on a real device — simulators don't replicate GPS |

- Every bug fix ships with a regression test.
- The order status-transition guard and the server-side totals calculation have dedicated unit tests — they are the highest-risk correctness surfaces.

---

## 9. Git & review

- Small, focused PRs. One concern per PR.
- Conventional-commit-style messages: `feat(orders): reject illegal status transitions`.
- CI gate: `tsc --noEmit` + lint + tests must pass before merge.
- Migrations are forward-only; never edit a shipped migration file — add a new one.
- No secrets in history. If a secret is ever committed, rotate it immediately and scrub — treat the key as compromised.

---

## 10. Definition of done

A change is done when:
1. `tsc --noEmit` passes in every affected package.
2. Lint passes.
3. Unit tests (and integration tests for backend routes) pass.
4. Location/payment changes verified on a real device / in Stripe test mode.
5. Shared-type changes are reflected in every consumer and in the DB constraint.
6. Docs updated if the API surface, schema, or a core rule changed.
