# Queup

**Skip the queue. Tap, pay, collect.**

Queup is a UK mobile ordering platform for food trucks, market stalls, festival vendors, home food businesses, and pop-up caterers. Customers find nearby vendors using real-time location and what3words, order, pay via Stripe, and collect using a 4-character code — no queuing, no cash, no miscommunication.

---

## Products

| Product | Platform | Audience |
|---|---|---|
| Customer app | iOS + Android | People buying food |
| Vendor app | iOS + Android | Food businesses selling |
| Manager dashboard | Web | Platform admin + event organisers |

---

## Quick start

```bash
# Install all dependencies
npm install

# Start backend API
npm run dev:backend

# Start customer app
npm run dev:customer

# Start vendor app
npm run dev:vendor

# Start manager dashboard
npm run dev:manager
```

---

## Environment setup

Copy `.env.example` to `.env` in each package and fill in:

- `SUPABASE_URL` and `SUPABASE_SERVICE_KEY` — from your Supabase project settings
- `STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET` — from Stripe dashboard
- `W3W_API_KEY` — from developer.what3words.com
- `FCM_SERVER_KEY` — from Firebase console

---

## Documentation

All project documentation is in `/docs`. Each folder is a workstream with its own dedicated Claude chat.

```
docs/
├── 00-master/           Project index and Claude instructions
├── 01-business-model/   Revenue model and projections
├── 02-customer-journey/ Screen-by-screen user flows
├── 03-design-ux/        Figma structure and design system
├── 04-technical/        Architecture, API, database
├── 05-security/         Auth, GDPR, PCI
├── 06-integrations/     Stripe, what3words, Firebase, Supabase
├── 07-legal/            Terms, privacy, trademark
├── 08-marketing/        Brand, GTM, launch plan
├── 09-costs/            Build and running costs
└── 10-implementation/   Sprint plan and App Store submission
```

---

## Team

- **Development**: You + Claude Code
- **UX / UI Design**: Your wife (Figma)
- **Product**: Both

---

## Stack

React Native + Expo · Node.js + Fastify · Supabase · Stripe Connect · Firebase · what3words · TypeScript throughout
