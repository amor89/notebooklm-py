# Queup — Claude Project Instructions

## About this project

Queup is a UK mobile food ordering platform with three products:
- **Customer app** (iOS + Android) — find food, order, pay, collect
- **Vendor app** (iOS + Android) — manage orders, location, menu, payouts
- **Manager dashboard** (web) — platform admin and event organiser tools

The platform serves food trucks, market stalls, festival vendors, home food businesses, and pop-up caterers across the UK.

---

## Documentation structure

All project documentation lives in `/docs` organised by workstream. Each workstream has a dedicated chat. Documents are markdown unless stated otherwise.

```
docs/
├── 00-master/          Project index, instructions, glossary
├── 01-business-model/  Revenue model, pricing, P&L
├── 02-customer-journey/ User flows, screen journeys, wireframe specs
├── 03-design-ux/       Figma structure, design system, components
├── 04-technical/       Architecture, API, database, code
├── 05-security/        Auth, GDPR, PCI, data protection
├── 06-integrations/    Stripe, what3words, Firebase, Supabase
├── 07-legal/           Terms, privacy, trademark, IP
├── 08-marketing/       Brand, GTM, launch plan, channels
├── 09-costs/           Build costs, running costs, unit economics
└── 10-implementation/  Sprint plan, milestones, App Store submission
```

---

## Tech stack

- Mobile: React Native + Expo (SDK 51+)
- Backend: Node.js + Fastify + TypeScript
- Database: Supabase (PostgreSQL + PostGIS)
- Auth: Supabase Auth
- Payments: Stripe + Stripe Connect
- Push notifications: Firebase Cloud Messaging
- Location: what3words API
- Manager web: React + Vite
- Monorepo: npm workspaces

---

## Three user types

**Customer** — downloads the app, signs up, finds food nearby, orders, pays, collects using a 4-character code.

**Vendor** — downloads the vendor app, onboards with Stripe Connect, manages their menu, broadcasts their location, receives and fulfils orders.

**Manager** — accesses the web dashboard to oversee vendors, view platform analytics, and manage event configurations.

---

## Core domain rules

- Prices always in pence (integers). `850` = £8.50. Use `formatPrice()` to display.
- Order status: `pending → accepted → preparing → ready → collected`. One direction only.
- Collection codes: 4-char alphanumeric, no 0/O/1/I. Generated server-side.
- Location updates: every 30 seconds from vendor app. One what3words API call per update.
- Platform fee: 5% via Stripe Connect `application_fee_amount`.
- Nearby search: PostGIS `vendors_within_radius` RPC, default 500m radius.
- Two customer-facing segments: **Segment A** (trucks/stalls, map discovery) and **Segment B** (home food, pre-order slots).

---

## Customer app — confirmed screen flow

1. Splash / onboarding (3 slides)
2. Auth — login or sign up (email, Google, Apple)
3. GPS permission prompt
4. Segment selector — Food Trucks / Home Food
5. Discovery screen — map (what3words), search bar, filters, vendor list with distance and ETA, floating basket button
6. Vendor profile — menu, prices, info
7. Item detail — add to basket
8. Basket review
9. Checkout — Stripe payment
10. Order confirmation — collection code
11. Order status tracker — live updates
12. Profile / account screen

---

## Vendor app — confirmed screen flow

1. Onboarding — business type, mode selection
2. Stripe Connect setup
3. Menu builder
4. Go Live screen — activates GPS broadcast, shows w3w address
5. Order queue — live incoming orders
6. Order detail — accept, prepare, mark ready
7. Payouts summary

---

## Manager dashboard — confirmed sections

1. Vendor management — approve, suspend, view
2. Order overview — platform-wide activity
3. Event configuration — set up geofenced event zones
4. Analytics — revenue, orders, active vendors
5. Support tools — refunds, disputes

---

## Location type by vendor segment

Location is handled differently depending on vendor mode. This is a core data rule that applies across all code, design, and documentation.

| Vendor mode | Location type | How it works |
|---|---|---|
| `festival` | what3words (dynamic) | GPS broadcast every 30s, converted to w3w address |
| `fixed_pitch` | what3words (dynamic) | GPS broadcast every 30s, converted to w3w address |
| `popup` | what3words (dynamic) | GPS broadcast every 30s, converted to w3w address |
| `home_food` | Fixed postal address | Street address + postcode entered at onboarding. Never changes. |

**Segment A vendors** (festival, fixed_pitch, popup) use what3words because they trade in locations with no reliable street address — open fields, car parks, market squares, festival sites.

**Segment B vendors** (home_food) use a fixed UK postal address with full street address and postcode. Customers see the street address. Distance is calculated from postcode centroid. No GPS broadcast required for home food vendors.

**Database implications:**
- `vendors` table has both `w3w_address` (nullable) and `postal_address` (nullable) columns
- `location_type` column stores `'w3w'` or `'postal'`
- Only Segment A vendors have background location active in the vendor app
- Segment B vendors set their address once at onboarding and it does not change

**Customer-facing implications:**
- Segment A vendor cards show: w3w badge + "Take me there" button
- Segment B vendor cards show: street address + postcode + distance in miles from customer's postcode

---

## Non-negotiable rules

- Never commit `.env` files
- Always validate Stripe webhook signatures
- Background location requires `expo-location` background permission in `app.json`
- PostGIS must be enabled in Supabase before migrations
- Never call what3words more than once per location update
- All documents in this project are markdown unless explicitly requested otherwise

---

## How to use this in a new chat

Start the message with:
> "I am working on the [WORKSTREAM NAME] chat for Queup."

Then attach:
1. This file
2. The workstream README from the relevant `/docs` subfolder
3. Any dependency documents listed in that README

When the chat produces output, save it as a `.md` file in the correct folder.
