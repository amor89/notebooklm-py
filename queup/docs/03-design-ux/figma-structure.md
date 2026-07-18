# Queup — Design & UX

## Workstream: 03-design-ux
## Chat purpose: Define Figma file structure, design system, component library, and screen specifications

### Dependencies
- `00-master/README.md`
- `00-master/claude-project-instructions.md`
- `02-customer-journey/customer-app-flow.md`

### Outputs expected from this chat
- `03-design-ux/figma-structure.md` (this file)
- `03-design-ux/design-system.md`
- `03-design-ux/component-library.md`
- `03-design-ux/screen-specs.md`

---

## Figma File Structure

Organise the Queup Figma workspace into one project with four files:

```
Queup (Figma Project)
├── 01 — Design System
├── 02 — Customer App
├── 03 — Vendor App
└── 04 — Manager Dashboard
```

---

## File 01 — Design System

### Pages inside this file:

**Page: Foundations**
- Colour palette (primary, secondary, neutrals, semantic colours)
- Typography scale (headings, body, captions, labels)
- Spacing system (4px base grid)
- Border radius tokens
- Shadow tokens
- Icon set

**Page: Components**
- Buttons (primary, secondary, ghost, danger — all states: default, hover, pressed, disabled, loading)
- Input fields (default, focused, error, disabled)
- Cards (vendor card, order card, item card)
- Bottom sheets
- Navigation bar (tab bar for mobile)
- Top bar / header
- Badges (dietary tags, status, distance)
- Modals and alerts
- Progress indicators
- Collection code display
- w3w address badge
- Map pin variants (by category)

**Page: Patterns**
- Order status flow pattern
- Empty states (no vendors nearby, empty basket, no orders)
- Loading states (skeleton screens)
- Error states

---

## Colour palette (proposed — your wife to confirm)

| Token | Value | Usage |
|---|---|---|
| `brand-primary` | #FF5C00 | Primary buttons, active states, brand |
| `brand-secondary` | #FFB800 | Accents, highlights |
| `surface-bg` | #0D0D0D | App background (dark mode) |
| `surface-card` | #1A1A1A | Card backgrounds |
| `surface-input` | #242424 | Input backgrounds |
| `text-primary` | #F0F0F0 | Body text |
| `text-secondary` | #888888 | Secondary labels |
| `text-disabled` | #444444 | Disabled states |
| `success` | #00C896 | Ready status, confirmations |
| `warning` | #FFB800 | Preparing status |
| `error` | #FF3B30 | Error states, rejection |
| `info` | #3B82F6 | Informational |

---

## Typography (proposed)

| Token | Font | Size | Weight | Usage |
|---|---|---|---|---|
| `display-xl` | Syne | 32px | 800 | Screen titles |
| `display-lg` | Syne | 24px | 700 | Section headings |
| `display-md` | Syne | 18px | 700 | Card titles |
| `body-lg` | DM Sans | 16px | 400 | Body text |
| `body-md` | DM Sans | 14px | 400 | Secondary text |
| `body-sm` | DM Sans | 12px | 400 | Captions, tags |
| `label` | DM Mono | 11px | 500 | Badges, codes, labels |
| `code` | DM Mono | 24px | 700 | Collection code display |

---

## File 02 — Customer App

### Pages inside this file:

**Page: User Flows**
- Complete flow diagram from splash to order collected
- Branching paths (GPS denied, auth errors, payment failure)

**Page: Splash + Onboarding**
- Splash screen
- Onboarding slide 1
- Onboarding slide 2
- Onboarding slide 3

**Page: Auth**
- Login screen
- Sign up screen
- Forgot password screen
- Email verification prompt

**Page: Permissions**
- GPS permission screen
- GPS denied fallback screen

**Page: Segment Selector**
- Segment selection screen (Food Trucks / Home Food)

**Page: Discovery — Segment A**
- Discovery screen (map + list, default state)
- Discovery screen (filter applied)
- Discovery screen (search active)
- Map pin states (default, selected, visited)

**Page: Vendor Profile**
- Vendor profile screen (top)
- Vendor profile screen (menu scroll)
- Item detail sheet

**Page: Basket + Checkout**
- Basket review screen
- Checkout screen (payment)
- Payment processing state

**Page: Order**
- Order confirmation screen
- Order status tracker — each state
- Order ready full-screen

**Page: Profile**
- Profile / account screen
- Order history screen

**Page: Prototype**
- Linked prototype for all flows ready for handoff and usability testing

---

## File 03 — Vendor App

### Pages inside this file:

**Page: User Flows**
**Page: Onboarding**
**Page: Stripe Setup**
**Page: Menu Builder**
**Page: Go Live**
**Page: Order Queue**
**Page: Order Detail**
**Page: Payouts**
**Page: Prototype**

---

## File 04 — Manager Dashboard

### Pages inside this file:

**Page: User Flows**
**Page: Login**
**Page: Vendor Management**
**Page: Order Overview**
**Page: Event Configuration**
**Page: Analytics**
**Page: Support Tools**
**Page: Prototype**

---

## Figma to code handoff protocol

When a screen is approved, your wife exports from Figma using the following process:

1. Mark the frame as "Ready for dev" using Figma's Dev Mode status
2. Ensure all layers are named with the component naming convention (e.g. `VendorCard/Default`, `Button/Primary/Active`)
3. All colours reference design tokens, not hardcoded hex values
4. All text uses text styles, not local overrides
5. Export measurements in points (not pixels) for React Native compatibility

You access the designs in Dev Mode and extract:
- Component dimensions and spacing
- Colour tokens (map to your existing CSS variables)
- Typography styles
- Asset exports (SVG for icons, PNG 2x/3x for photos)

---

## Screen size targets

| Device | Screen size | Priority |
|---|---|---|
| iPhone 14 / 15 | 390 × 844pt | Primary iOS |
| iPhone SE | 375 × 667pt | Small iOS |
| Pixel 7 | 412 × 915pt | Primary Android |
| Samsung Galaxy S23 | 360 × 780pt | Secondary Android |

Design at 390 × 844pt. Test at 375pt width to ensure nothing breaks on smaller screens.

---

## Accessibility requirements

- Minimum touch target: 44 × 44pt
- Colour contrast: WCAG AA minimum (4.5:1 for normal text, 3:1 for large text)
- All icons have accessible labels
- All interactive elements have focus states
- Font sizes respect system accessibility settings (use Expo's `useWindowDimensions` hook)
