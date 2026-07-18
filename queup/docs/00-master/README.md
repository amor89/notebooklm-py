# Queup — Master Project Index

## What Queup is

Queup is a UK mobile ordering platform for itinerant food businesses and home food sellers. Customers find nearby vendors, order, pay, and collect — no queuing, no cash, no miscommunication. Vendors manage orders from their phone and get paid directly via Stripe Connect.

The name "Queup" directly addresses the core problem: eliminating the queue. It works for every segment — festival food trucks, fixed pitch vendors, home bakers, market stalls, and pop-up caterers.

---

## Three products

| Product | Who uses it | Platform |
|---|---|---|
| **Queup Customer App** | People buying food | iOS + Android (React Native + Expo) |
| **Queup Vendor App** | Food businesses selling | iOS + Android (React Native + Expo) |
| **Queup Manager Dashboard** | Platform admin, event organisers | Web (React) |

---

## How to use this documentation

Each folder in `/docs` corresponds to one project workstream. Each workstream has its own dedicated Claude chat. When you complete work in one chat, export the output as a markdown file and attach it to the relevant folder here.

Every chat should begin by attaching:
1. This file (`00-master/README.md`)
2. The relevant workstream file for that chat topic
3. Any completed documents from other chats that are referenced

This keeps all chats connected to a single source of truth.

---

## Documentation index

| Folder | Topic | Chat status |
|---|---|---|
| `00-master/` | This index, project instructions, glossary | Active |
| `01-business-model/` | Revenue model, pricing, P&L projections | Pending |
| `02-customer-journey/` | All user flows, screen-by-screen journeys | Active |
| `03-design-ux/` | Figma structure, design system, component library | Pending |
| `04-technical/` | Architecture, API, database, code standards | Complete |
| `05-security/` | Auth, data protection, GDPR, PCI compliance | Complete |
| `06-integrations/` | Stripe, what3words, Firebase, Supabase, Figma API | Pending |
| `07-legal/` | Terms of service, privacy policy, trademark, IP | Pending |
| `08-marketing/` | Go-to-market, brand, channels, launch plan | Pending |
| `09-costs/` | Build costs, running costs, unit economics | Pending |
| `10-implementation/` | Sprint plan, milestones, App Store submission | Pending |

---

## Team

| Role | Person |
|---|---|
| Development + Claude Code | You |
| UX / UI Design + Figma | Your wife |
| Product decisions | Both |

---

## Core technology decisions

| Decision | Choice | Reason |
|---|---|---|
| Mobile framework | React Native + Expo | One codebase for iOS and Android |
| Backend | Node.js + Fastify | Fast, TypeScript-native |
| Database | Supabase (PostgreSQL + PostGIS) | Geospatial queries, auth, realtime built in |
| Payments | Stripe + Stripe Connect | PCI compliant, vendor payouts handled |
| Location encoding | what3words API | Works in fields with no street address |
| Push notifications | Firebase Cloud Messaging | Free at MVP scale, iOS + Android |
| Manager dashboard | React (Vite) | Lightweight, fast to build |
| Monorepo | npm workspaces | Shared types across all three products |

---

## Glossary

| Term | Definition |
|---|---|
| Vendor | A food business on the platform (truck, home cook, stall) |
| Customer | A person buying food via Queup |
| Manager | Platform admin or event organiser using the web dashboard |
| Pitch | The location where a vendor trades |
| w3w address | A what3words three-word location (e.g. `///filled.count.soap`) |
| Postal address | A fixed UK street address + postcode used by home food vendors |
| Collection code | A 4-character code the customer shows to collect their order |
| Vendor mode | One of: `festival`, `fixed_pitch`, `home_food`, `popup` |
| Location type | `w3w` for Segment A vendors; `postal` for Segment B (home food) vendors |
| Order status | `pending → accepted → preparing → ready → collected` |
| Platform fee | 5% of order total taken via Stripe Connect |
| Segment A | festival, fixed_pitch, popup — use what3words, real-time GPS |
| Segment B | home_food — use fixed postal address, pre-order slot model |

---

## Non-negotiable rules (applies to all chats)

- All prices stored as pence (integers). Never floats.
- Order status flows one direction only. No reversals except rejection.
- Stripe webhook signatures must always be validated.
- No `.env` files committed to git. Ever.
- what3words called once per location update only — Segment A vendors only.
- Home food vendors (Segment B) use a fixed postal address. Never GPS. Never what3words.
- PostGIS must be enabled in Supabase before running migrations.
- Background location permission required in vendor app `app.json` for Segment A vendors only.
- All output documents in this project are markdown unless explicitly requested otherwise.

---

## Document attachment protocol

When starting a new chat for a workstream:

1. Attach `00-master/README.md` (this file)
2. Attach the workstream's own index file (e.g. `02-customer-journey/README.md`)
3. Attach any completed documents flagged as dependencies
4. State at the top of the chat: "I am working on the [workstream name] chat for Queup."

When a chat produces a document, save it as a `.md` file in the correct folder and update the status column in the index table above.
