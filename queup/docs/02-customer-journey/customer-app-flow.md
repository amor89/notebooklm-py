# Queup — Customer Journey

## Workstream: 02-customer-journey
## Chat purpose: Define every screen, interaction, and decision point for the customer app

### Dependencies
- `00-master/README.md`
- `00-master/claude-project-instructions.md`

### Outputs expected from this chat
- `02-customer-journey/customer-app-flow.md` (this file)
- `02-customer-journey/vendor-app-flow.md`
- `02-customer-journey/manager-dashboard-flow.md`
- `02-customer-journey/edge-cases.md`

---

## Part 1 — Customer App Journey

### Overview

The customer downloads Queup from the App Store or Google Play, creates an account, grants location access, selects their mode (food truck discovery or home food pre-order), and browses, orders, pays, and collects food.

---

### Screen 1 — Splash Screen

**Purpose:** Brand moment on first load.

- Queup logo centred
- Tagline: *Skip the queue. Tap, pay, collect.*
- Duration: 2 seconds, then auto-advances
- No user action required

---

### Screen 2 — Onboarding (3 slides)

**Purpose:** Communicate the value proposition before sign-up.

**Slide 1**
- Headline: Find food near you
- Body: Discover food trucks, market stalls, and home cooks wherever you are
- Visual: Map with vendor pins

**Slide 2**
- Headline: Order without the queue
- Body: Browse menus, pay securely, and get notified when your food is ready
- Visual: Phone showing order confirmation and collection code

**Slide 3**
- Headline: Just show your code
- Body: Walk up, show your 4-letter code, and collect
- Visual: Collection code display

**Navigation:** Swipeable slides. Skip button top right. Get Started button on final slide.

---

### Screen 3 — Authentication

**Purpose:** Create or access an account.

**Options presented:**
- Continue with Apple (iOS only)
- Continue with Google
- Continue with email

**Sign up flow (email):**
1. Enter email address
2. Enter password (min 8 characters, one number)
3. Enter full name
4. Tap Create Account
5. Verification email sent — prompt to check inbox
6. On verification, advance to GPS permission screen

**Log in flow (email):**
1. Enter email
2. Enter password
3. Forgot password link → email reset flow
4. Tap Log In → advance to GPS permission (or discovery screen if already granted)

**Social auth (Google / Apple):**
1. Tap provider button
2. System OAuth sheet appears
3. On success, if new user → GPS permission screen
4. If returning user → discovery screen

**Error states:**
- Email already registered → show "Log in instead?" prompt
- Wrong password → "Incorrect password. Try again or reset."
- No network → "No connection. Check your internet and try again."

---

### Screen 4 — GPS Permission

**Purpose:** Request location access. Required for the core discovery feature.

**Copy:**
- Headline: *Find food near you*
- Body: *Queup uses your location to show you vendors within walking distance. We never share your location with anyone.*
- Button: Allow Location Access

**System permission prompt fires after tapping the button.**

**If denied:**
- Show explanation screen: "Without location access, you won't see vendors near you. You can enable it in Settings > Queup > Location."
- Two options: Open Settings / Continue without location (manual postcode search only)

---

### Screen 5 — Segment Selector

**Purpose:** Route the customer to the correct discovery experience.

**Two tiles presented full-screen:**

**Tile 1 — Food Trucks & Stalls**
- Icon: food truck illustration
- Label: Food Trucks & Stalls
- Description: Find vendors near you right now — festivals, markets, and street pitches
- Tapping advances to the map-based discovery screen

**Tile 2 — Home Food**
- Icon: house / kitchen illustration
- Label: Home Food
- Description: Pre-order from local home cooks and bakers — pick a date and collect
- Tapping advances to the home food browse screen (pre-order slot model)

**Note:** User preference is saved. On next open, app defaults to last used segment. Segment switcher is accessible from discovery screen.

---

### Screen 6A — Discovery Screen (Segment A: Food Trucks & Stalls)

**Purpose:** Core discovery experience. Find vendors nearby in real time.

**Layout (top to bottom):**

**Top bar**
- Search bar: "Search food, cuisine, or vendor name"
- Filter icon (right of search) — opens filter bottom sheet
- Profile icon (right of filter) — opens personal account screen

**Map section (top half of screen)**
- Interactive map using what3words grid overlay
- Vendor pins on map — tapping a pin opens vendor card
- User location dot
- Recenter button (bottom right of map)
- Each vendor pin shows category icon (burger, pizza, etc.)

**Vendor list (bottom half, scrollable)**
- Cards sorted by distance (nearest first)
- Each card shows:
  - Vendor name and category
  - Distance (metres under 1000m, then miles)
  - Estimated wait time (pulled from vendor's current queue length)
  - Dietary tags (vegan, halal, gluten-free) if applicable
  - Rating (once review system is implemented — Phase 2)
  - w3w address badge (e.g. `///filled.count.soap`)
  - Thumbnail image

**Floating basket button (centre bottom)**
- Shows basket item count badge when items are added
- Always visible when basket has items
- Tapping opens basket review screen

**Filter bottom sheet options:**
- Cuisine type (multi-select): Burgers, Pizza, Asian, Mexican, Vegan, Desserts, Drinks, Other
- Dietary: Vegan, Vegetarian, Gluten-free, Halal, Dairy-free
- Distance radius: 250m / 500m / 1km / 2km
- Open now only (toggle)

---

### Screen 6B — Discovery Screen (Segment B: Home Food)

**Purpose:** Browse home food businesses available for pre-order.

**Location type:** Home food vendors use a fixed postal address (street address + postcode), not GPS or what3words. Distance shown to customer is calculated from their postcode to the vendor's postcode centroid, displayed in miles.

**Layout:**
- Search bar: "Search home cooks, bakers, cuisine"
- Category filters (Baked goods, Meal prep, Desserts, World food, etc.)
- List of home food vendors showing:
  - Vendor name and speciality
  - Street address and postcode (full address — customer needs this to collect)
  - Distance in miles from customer's location
  - Next available collection date
  - Price range indicator
  - Dietary tags

**Pre-order flow:**
1. Customer selects vendor
2. Views menu with available collection dates and time slots
3. Selects date and time slot
4. Adds items to basket
5. Pays upfront via Stripe
6. Receives confirmation with full street address and collection code
7. On order day: push notification reminder 2 hours before slot

**Key difference from Segment A:**
- No map view — home food uses address list only
- No real-time availability — based on vendor-defined slots
- Collection address is a fixed street address, not a w3w location
- Distance is miles (not metres) and approximate (postcode-based)

---

### Screen 7 — Vendor Profile

**Purpose:** Show full vendor information and menu before ordering.

**Location display — depends on vendor mode:**

For Segment A vendors (festival, fixed_pitch, popup):
- w3w address badge (e.g. `///filled.count.soap`) with "Take me there" button
- Estimated wait time based on current queue

For Segment B vendors (home_food):
- Full street address and postcode
- Next available collection slots
- No estimated wait time — slot-based only

**Sections (all vendors):**
- Hero image (vendor photo or food photo)
- Vendor name, category, rating
- Location display (see above — differs by vendor mode)
- Dietary tags
- Menu — grouped by category (Mains, Sides, Drinks, Extras)
- Each menu item: name, description, price, dietary icons, photo (if available)
- Out-of-stock items shown greyed with "Unavailable" label

**Add to basket:**
- Tapping an item opens item detail screen

---

### Screen 8 — Item Detail

**Purpose:** Confirm item selection and any customisation.

**Shows:**
- Item photo (full width)
- Item name and price
- Description
- Dietary information
- Quantity selector (+ / -)
- Special instructions text field (optional, 100 char limit)
- Add to Basket button (shows running total)

---

### Screen 9 — Basket Review

**Purpose:** Review all items before checkout.

**Shows:**
- List of items with quantities and prices
- Edit / remove per item
- Subtotal
- Platform service charge (displayed as "Service charge: £X.XX")
- Total
- Vendor name reminder
- Location reminder — w3w address (Segment A) or street address (Segment B)
- Proceed to Checkout button
- Continue Shopping button

---

### Screen 10 — Checkout

**Purpose:** Collect payment.

**Flow:**
1. Review order summary (collapsed)
2. Stripe payment sheet appears:
   - Saved cards (returning users)
   - Add new card
   - Apple Pay / Google Pay (where available)
3. Tap Pay £X.XX
4. Stripe processes payment
5. On success → Order Confirmation screen
6. On failure → Error message with retry option

**Note:** Payment is taken immediately. Orders are confirmed only after payment succeeds.

---

### Screen 11 — Order Confirmation

**Purpose:** Reassure customer and give them what they need to collect.

**Shows:**
- Large collection code (e.g. **Q7MF**) — tap to copy
- Vendor name
- Location — w3w address with "Take me there" button (Segment A) or full street address + postcode (Segment B)
- Estimated wait time (Segment A only) or collection slot time (Segment B)
- Order summary (collapsible)
- "Track my order" button → advances to order status screen

---

### Screen 12 — Order Status Tracker

**Purpose:** Live status updates while the vendor prepares the order.

**Status steps shown as a progress bar:**
1. Order received
2. Accepted by vendor
3. Being prepared
4. Ready to collect ✓

**Each step shows timestamp when reached.**

**Push notification fires at each status change.**

**When Ready:**
- Full-screen highlight: "Your order is ready!"
- Large collection code displayed again
- Location — w3w address with "Take me there" button (Segment A) or full street address (Segment B)
- Collect button (marks order as collected when tapped)

---

### Screen 13 — Profile / Account Screen

**Purpose:** Personal account management.

**Sections:**
- Name and email
- Order history (last 10 orders, link to full history)
- Payment methods (managed via Stripe)
- Notification preferences
- Location settings shortcut
- Help and support
- Log out
- Delete account

---

## Part 2 — Vendor App Journey

*Full flow to be documented in `02-customer-journey/vendor-app-flow.md`*

**Confirmed screens (outline):**
1. Onboarding — business name, mode selection (festival / fixed pitch / home food / popup)
2. Stripe Connect setup — bank account, identity verification
3. Menu builder — add items, set prices, toggle availability
4. Go Live — activates GPS broadcast, displays w3w address in large text
5. Order queue — incoming orders in real time
6. Order detail — accept, start preparing, mark ready
7. Payout summary — earnings, transfer history

---

## Part 3 — Manager Dashboard Journey

*Full flow to be documented in `02-customer-journey/manager-dashboard-flow.md`*

**Confirmed sections (outline):**
1. Login — email and password, 2FA
2. Vendor management — approve new vendors, suspend, view profiles
3. Order overview — live order feed across all vendors
4. Event configuration — create event zones with geofence, assign vendors
5. Analytics — revenue, GMV, active vendors, orders per day
6. Support tools — issue refunds, handle disputes, contact vendors

---

## Edge cases to document

*To be completed in `02-customer-journey/edge-cases.md`*

- Vendor goes offline mid-order
- Payment succeeds but order creation fails
- Customer attempts to order from vendor with empty menu
- GPS denied — fallback to postcode search
- App backgrounded during checkout
- Vendor rejects order — customer refund flow
- Collection code used twice
- Order marked collected but customer disputes non-receipt
