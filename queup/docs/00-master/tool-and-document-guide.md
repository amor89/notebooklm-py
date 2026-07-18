# Queup — Tool & Document Guide Per Workstream

This is the reference for every chat in the Queup project. It tells you which Claude tools to use, which documents to attach, which documents to produce, and which documents from other chats to bring in as context.

---

## How every chat starts

At the top of every new chat, write this line:
> "I am working on the [WORKSTREAM NAME] chat for Queup."

Then attach the files listed in the "Attach at start" column below.

All output documents are markdown (`.md`) unless you explicitly ask for something else.

When a document is complete, save it to the correct folder and update the status in `00-master/README.md`.

---

## Location type rule (applies to every chat)

| Vendor mode | Location |
|---|---|
| festival, fixed_pitch, popup | what3words (dynamic GPS, Segment A) |
| home_food | Fixed postal address with street and postcode (Segment B) |

Never use what3words for home food vendors. Never use GPS or what3words in Segment B code, designs, or documentation.

---

## Workstream 00 — Master (this project chat)

**Purpose:** Maintain the source of truth. Update the index when statuses change. Amend core rules when decisions are made.

| | |
|---|---|
| **Claude tools** | Artifacts, file creation |
| **Attach at start** | Nothing — this is the source |
| **Produces** | `00-master/README.md`, `00-master/claude-project-instructions.md`, this file |
| **Depends on** | Nothing |
| **When to update** | Any time a core decision changes (location type, pricing, naming, tech stack) |

---

## Workstream 01 — Business Model

**Purpose:** Define how Queup makes money, model revenue at different scales, and produce a 3-year P&L.

| | |
|---|---|
| **Claude tools** | Artifacts (tables, charts), file creation, web search (UK market data) |
| **Attach at start** | `00-master/README.md`, `00-master/claude-project-instructions.md`, `01-business-model/README.md` |
| **Produces** | `01-business-model/revenue-model.md`, `01-business-model/pricing-strategy.md`, `01-business-model/unit-economics.md`, `01-business-model/projections-3yr.md` |
| **Depends on** | Nothing — this is upstream of most other workstreams |
| **Bring in later** | Attach `01-business-model/revenue-model.md` to chats 03, 08, 09 |

**Key questions for this chat:**
- What commission rate makes the unit economics work?
- When does a subscription tier make sense?
- What is the break-even vendor count?
- 3-year projection at 0.5%, 1%, and 2% market penetration

---

## Workstream 02 — Customer Journey

**Purpose:** Define every screen, state, interaction, and edge case for all three products.

| | |
|---|---|
| **Claude tools** | Artifacts (flow diagrams), file creation |
| **Attach at start** | `00-master/README.md`, `00-master/claude-project-instructions.md`, `02-customer-journey/customer-app-flow.md` |
| **Produces** | `02-customer-journey/customer-app-flow.md` ✓ (complete), `02-customer-journey/vendor-app-flow.md`, `02-customer-journey/manager-dashboard-flow.md`, `02-customer-journey/edge-cases.md` |
| **Depends on** | `00-master/README.md` |
| **Bring in later** | Attach all journey files to chats 03, 04, 05, 10 |

**Still to complete in this chat:**
- Full vendor app journey (7 screens)
- Full manager dashboard journey (5 sections)
- Edge cases document (8+ scenarios)
- Home food segment journey (Segment B — now confirmed: postal address, pre-order slots)

**Location rule reminder:**
- Segment A screens show w3w badge and "Take me there"
- Segment B screens show street address, postcode, and distance in miles

---

## Workstream 03 — Design & UX

**Purpose:** Define Figma file structure, design system, component library, and screen specs for your wife to build from.

| | |
|---|---|
| **Claude tools** | Artifacts (visual mockups, component specs), file creation |
| **Attach at start** | `00-master/README.md`, `00-master/claude-project-instructions.md`, `02-customer-journey/customer-app-flow.md`, `03-design-ux/figma-structure.md` |
| **Produces** | `03-design-ux/figma-structure.md` ✓ (complete), `03-design-ux/design-system.md`, `03-design-ux/component-library.md`, `03-design-ux/screen-specs.md` |
| **Depends on** | `02-customer-journey/customer-app-flow.md` must be complete first |
| **Bring in later** | Attach `03-design-ux/design-system.md` to chat 04 for CSS token mapping |

**Location rule in design:**
- Segment A vendor card component: w3w badge, "Take me there" button, distance in metres/miles
- Segment B vendor card component: street address text, postcode, distance in miles
- Two variants of the vendor card component are required in the Figma component library

**Your wife's primary reference files:**
1. `03-design-ux/figma-structure.md` — how to organise the Figma workspace
2. `02-customer-journey/customer-app-flow.md` — screen-by-screen content to design

---

## Workstream 04 — Technical Architecture

**Purpose:** Complete the backend, build both mobile apps, and build the manager dashboard using Claude Code.

| | |
|---|---|
| **Claude tools** | Claude Code (primary), file creation, bash |
| **Attach at start** | `00-master/README.md`, `CLAUDE.md`, `04-technical/README.md`, `02-customer-journey/customer-app-flow.md` |
| **Produces** | `04-technical/architecture.md`, `04-technical/api-reference.md`, `04-technical/database-schema.md`, `04-technical/code-standards.md` |
| **Depends on** | `02-customer-journey/` flows should be complete before building screens |
| **Bring in later** | Attach `04-technical/database-schema.md` to chats 05 and 06 |

**CLAUDE.md is the key file for Claude Code.** It lives in the repo root and is read automatically on every Claude Code session. Keep it under 100 lines.

**Location rule in code:**
- `vendors` table needs: `location_type VARCHAR` ('w3w' or 'postal'), `w3w_address VARCHAR` (nullable), `postal_address JSONB` (nullable, contains line1, line2, city, postcode)
- Background location service in vendor app only activates when `vendor.location_type === 'w3w'`
- `geosearch.service.ts` uses PostGIS for Segment A (coordinate-based), postcode distance for Segment B

**Sub-chats recommended for this workstream:**
- One chat for backend completion (notifications, auth middleware, migrations)
- One chat for vendor app screens
- One chat for customer app screens
- One chat for manager dashboard

---

## Workstream 05 — Security

**Purpose:** Design auth, document GDPR compliance, confirm PCI scope, and produce a security checklist.

| | |
|---|---|
| **Claude tools** | File creation, web search (ICO guidance, UK GDPR, PCI DSS documentation) |
| **Attach at start** | `00-master/README.md`, `00-master/claude-project-instructions.md`, `05-security/README.md`, `04-technical/database-schema.md` (when complete) |
| **Produces** | `05-security/auth-design.md`, `05-security/gdpr-compliance.md`, `05-security/pci-dss-notes.md`, `05-security/security-checklist.md` |
| **Depends on** | `04-technical/database-schema.md` — need to know what data is stored before writing GDPR doc |
| **Bring in later** | Attach `05-security/gdpr-compliance.md` to chat 07 (legal) |

**Location data and GDPR:**
- Segment A vendor GPS coordinates are processed transiently (not stored long-term — only current position)
- Segment B vendor postal addresses are stored permanently and classed as personal data
- Customer GPS coordinates are never stored — used only for real-time proximity query
- Both types must be covered in the privacy policy

---

## Workstream 06 — Integrations

**Purpose:** Write complete integration guides for every third-party service so you (or Claude Code) can implement them correctly.

| | |
|---|---|
| **Claude tools** | Web search (official docs), file creation |
| **Attach at start** | `00-master/README.md`, `06-integrations/README.md`, `04-technical/README.md` |
| **Produces** | `06-integrations/stripe-connect.md`, `06-integrations/what3words.md`, `06-integrations/firebase-fcm.md`, `06-integrations/supabase-postgis.md`, `06-integrations/expo-location.md` |
| **Depends on** | Nothing — write these guides early so chat 04 can reference them |
| **Bring in later** | Attach relevant integration files when building in chat 04 |

**what3words integration scope:**
- Used ONLY for Segment A vendors (festival, fixed_pitch, popup)
- API called server-side only — key never in client bundle
- Called once per 30-second location update
- `what3words.md` guide should cover: API setup, server-side call, error handling, rate limit monitoring

**Expo location scope:**
- Background location required for Segment A vendor app only
- Must document the `app.json` permission strings required for both iOS and Android
- Not needed at all for home food (Segment B) vendor app screens

---

## Workstream 07 — Legal

**Purpose:** Draft terms of service, privacy policy, vendor agreement, and research trademark protection.

| | |
|---|---|
| **Claude tools** | Web search (UK IPO, ICO, FCA guidance), file creation |
| **Attach at start** | `00-master/README.md`, `07-legal/README.md`, `05-security/gdpr-compliance.md` (when complete) |
| **Produces** | `07-legal/terms-of-service-draft.md`, `07-legal/privacy-policy-draft.md`, `07-legal/trademark-research.md`, `07-legal/ip-protection.md`, `07-legal/vendor-agreement.md` |
| **Depends on** | `05-security/gdpr-compliance.md` for accurate privacy policy |
| **Note** | Outputs are drafts for solicitor review — not final legal documents |

**Location data in privacy policy:**
- Must explain that Segment A vendor GPS is broadcast live but not stored long-term
- Must explain that Segment B vendor postal address is stored permanently
- Must confirm that customer GPS is never stored
- Must cover right to erasure for both vendor types

---

## Workstream 08 — Marketing

**Purpose:** Build brand guidelines, go-to-market strategy, and launch plan.

| | |
|---|---|
| **Claude tools** | Web search (UK festival calendar, street food stats), file creation, image search (brand references) |
| **Attach at start** | `00-master/README.md`, `08-marketing/README.md`, `01-business-model/revenue-model.md` (when complete) |
| **Produces** | `08-marketing/brand-guidelines.md`, `08-marketing/go-to-market.md`, `08-marketing/launch-plan.md`, `08-marketing/vendor-acquisition.md`, `08-marketing/customer-acquisition.md` |
| **Depends on** | `01-business-model/revenue-model.md` — need pricing confirmed before marketing copy |

**Two distinct audiences for marketing:**
- **Vendors:** Food entrepreneurs who hate cash, queues, and WhatsApp DMs. Message: "Run your pitch like a pro."
- **Customers:** Festival-goers and food lovers who hate queuing. Message: "Skip the queue. Tap, pay, collect."

**Segment A vs Segment B in marketing:**
- Segment A: urgency and discovery — "Find it now. Order from the field."
- Segment B: trust and convenience — "Order ahead. Show up and collect."

---

## Workstream 09 — Costs

**Purpose:** Produce a detailed cost model for build phase, running costs, and break-even analysis.

| | |
|---|---|
| **Claude tools** | Artifacts (tables, financial models), file creation |
| **Attach at start** | `00-master/README.md`, `09-costs/README.md`, `01-business-model/unit-economics.md` (when complete) |
| **Produces** | `09-costs/build-costs.md`, `09-costs/running-costs.md`, `09-costs/unit-economics.md`, `09-costs/financial-model.md` |
| **Depends on** | `01-business-model/revenue-model.md` and `01-business-model/unit-economics.md` |

---

## Workstream 10 — Implementation

**Purpose:** Sprint plan, App Store submission checklist, beta test plan, and launch checklist.

| | |
|---|---|
| **Claude tools** | Artifacts (Gantt-style timelines, checklists), file creation |
| **Attach at start** | `00-master/README.md`, `10-implementation/README.md`, `04-technical/README.md`, `02-customer-journey/customer-app-flow.md` |
| **Produces** | `10-implementation/sprint-plan.md`, `10-implementation/milestone-tracker.md`, `10-implementation/app-store-submission.md`, `10-implementation/launch-checklist.md`, `10-implementation/beta-test-plan.md` |
| **Depends on** | Most other workstreams should be in progress before this is finalised |

---

## Summary table

| Chat | Primary tool | Attach at start | Key output |
|---|---|---|---|
| 00 Master | File creation | — | README, project instructions |
| 01 Business model | Artifacts + web search | 00-master files | Revenue model, P&L |
| 02 Customer journey | Artifacts + file creation | 00-master files | All user flows |
| 03 Design UX | Artifacts + file creation | 00-master + journey files | Figma structure, design system |
| 04 Technical | **Claude Code** | `CLAUDE.md` + journey files | Code, API, schema |
| 05 Security | Web search + file creation | 00-master + schema | GDPR doc, security checklist |
| 06 Integrations | Web search + file creation | 00-master + technical | Integration guides |
| 07 Legal | Web search + file creation | 00-master + GDPR doc | Terms, privacy policy |
| 08 Marketing | Web search + file creation | 00-master + revenue model | GTM, launch plan |
| 09 Costs | Artifacts + file creation | 00-master + unit economics | Cost model |
| 10 Implementation | Artifacts + file creation | 00-master + all complete docs | Sprint plan, launch checklist |
