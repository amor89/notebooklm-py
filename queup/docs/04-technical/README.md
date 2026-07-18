# Queup — Technical Architecture

## Workstream: 04-technical
## Chat purpose: Architecture decisions, API design, database schema, code standards, Claude Code usage

### Dependencies
- `00-master/README.md`
- `00-master/claude-project-instructions.md`
- `02-customer-journey/customer-app-flow.md`

### Outputs expected from this chat
- `04-technical/architecture.md` ✓ complete
- `04-technical/api-reference.md` ✓ complete
- `04-technical/database-schema.md` ✓ complete
- `04-technical/code-standards.md` ✓ complete
- `04-technical/claude-code-instructions.md` ✓ complete

### What is already built

The repository (`streetserve`, being renamed to `queup`) contains:
- Monorepo root with npm workspaces
- `shared/types/index.ts` — Vendor, MenuItem, Order, Customer, OrderStatus, LocationUpdate
- `shared/utils/` — formatPrice, generateCollectionCode
- `backend/src/index.ts` — Fastify entry point
- `backend/src/db/client.ts` — Supabase singleton
- `backend/src/routes/vendors.ts` — nearby vendors (PostGIS), vendor profile
- `backend/src/routes/orders.ts` — create, status update, notify
- `backend/src/routes/payments.ts` — Stripe PaymentIntent, webhook, Connect onboarding
- `backend/src/routes/location.ts` — GPS update with w3w conversion
- `backend/src/services/w3w.service.ts` — what3words API wrapper

### What still needs building

Backend:
- `notifications.service.ts` — Firebase Cloud Messaging
- `auth.middleware.ts` — JWT validation via Supabase
- `rateLimit.middleware.ts`
- All SQL migrations (vendors, menus, orders, customers, payment_confirmations)
- PostGIS `vendors_within_radius` RPC function
- `geosearch.service.ts`

Customer app (Expo):
- All screens listed in `02-customer-journey/customer-app-flow.md`
- `useLocationBroadcast` hook
- Stripe payment sheet integration
- Firebase push notification registration

Vendor app (Expo):
- All screens listed in vendor app flow
- Background location service
- Order queue with real-time Supabase subscription

Manager dashboard (React + Vite):
- All sections listed in manager flow

### Monorepo structure
```
queup/
├── apps/
│   ├── customer-app/    React Native + Expo
│   ├── vendor-app/      React Native + Expo
│   └── manager-web/     React + Vite
├── backend/             Node.js + Fastify
├── shared/              Types + utils (TypeScript)
└── docs/                All project documentation
```

### Code conventions
- TypeScript strict mode everywhere. No `any`.
- Named exports only. No default exports.
- Shared types in `shared/types/index.ts`, imported as `@queup/shared`.
- All env vars in `.env`. Never hardcoded. Never committed.
- Functional components + hooks only in React Native.
- All prices as pence (integers).

### Claude Code usage in this workstream
When using Claude Code to generate code:
1. Always reference the existing file structure before generating new files
2. Run `tsc --noEmit` after any TypeScript changes to catch type errors
3. Use `npx expo start` to test mobile changes — do not assume they work without running
4. For database changes, always generate a migration file, never edit schema directly
